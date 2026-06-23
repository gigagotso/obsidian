#  Conversation Reports — Permissions & User Filters: How It Works and What's Broken

A full walkthrough of how conversation reports are built, how permissions and
user-based filters shape the result set, and the exact reason the **Excel export**
produces different results from the **on-screen report** for the same user.

---

## 1. The two entry points (and why this matters)

Both the on-screen report and the Excel export go through the **same query builder**,
but in **two different runtime environments**. That difference is the whole bug.

| | On-screen report ("view") | Excel export |
|---|---|---|
| Controller | `ConversationReportController::index()` | `ConversationReportController::export()` |
| Runtime | **Synchronous** — the web (php-fpm) process handling the HTTP request | **Asynchronous** — a long-lived **queue worker** process |
| Auth | Real authenticated token user | Job manually calls `auth()->setUser($user)` |
| Builds query | `ConversationReportService::generate()` → `ConversationReport::report()` | same `generate()` → same `report()`, but inside `ConversationReportsExport::handle()` |

Source:
- `app/Http/Controllers/Reports/ConversationReportController.php:24` (`index`) and `:40` (`export`)
- `app/Jobs/ConversationReportsExport.php:50` (`handle`)
- `app/Services/Reports/ConversationReportService.php:19` (`generate`)

`export()` simply validates the **same** filters, then dispatches a queued job:

```php
// ConversationReportController.php:44
ConversationReportsExport::withChain([
    new NotifyUserOfCompletedExport($request->user(), $fileNameAndPath)
])->dispatch($this->lambdaExcelFuncName, $fileNameAndPath, $request->user(), $request->validated());
```

So **the query is identical**. The only thing that differs is *where* it runs and
*how permissions resolve there*. Hold that thought.

---

## 2. Request validation & authorization (the front door)

`ConversationReportRequest` (parent of `ExportConversationReportRequest`) gates access:

```php
// app/Http/Requests/Reports/ConversationReportRequest.php:19
public function authorize(): bool
{
    return auth()->user()->can('conversations:viewAny')
        || auth()->user()->can('conversations:view');
}
```

- `conversations:viewAny` → may see **all** conversations in the account.
- `conversations:view` → may only see conversations they **personally participated in**
  (enforced later by the permission scope).

It also normalizes input: default date range (last 7 days), default pagination,
and transforms the client filter shape into `filters.and[]`. The export request
just raises the page limit to 50,000 (`ExportConversationReportRequest.php:7,21`).

Validated output is reshaped into `date_range`, `pagination`, `filters`, `search`,
`sort_field`, `sort_direction` (`ConversationReportRequest.php:127`).

> **Key fact:** `authorize()` runs in HTTP, where permissions resolve correctly.
> Both view and export pass this gate identically. The divergence happens *later*,
> inside the query.

---

## 3. The query pipeline — `scopeReport()`

`app/Models/ConversationReport.php:237`. Order of operations:

```
getUserIdsFromParams()      → collect agent ids for name lookup (NOT permission scoped)
buildBaseQuery()            → select cr.* + joins (visitors, queues, widgets, conversations)
applyDateRange()            → WHERE cr.opened_at BETWEEN from AND to
applyAccountScope()         → WHERE cr.account_id = <user account_id>
applyPermissionScope()      → ★ the security filter (see §4)
applyFilters()              → user-chosen filters incl. user_ids (see §5)
applyGlobalSearch()         → optional full-text-ish search
applySortScope()            → ORDER BY
paginate(limit)             → page slice
... then bulk-load IPs, tags, reviews, recordings, user durations
```

Two of these are "user-based" but mean completely different things — this is the
part people conflate:

- **`applyPermissionScope()`** — a *security* restriction the system imposes based on
  the viewer's permissions. The user cannot bypass it.
- **`applyUserIdsFilter()`** — an *optional* filter the user *chooses* ("show me only
  agent X's conversations").

---

## 4. The permission scope (security — system-imposed)

```php
// app/Models/ConversationReport.php:473
protected function applyPermissionScope(Builder $query): void
{
    if (auth()->user()->can('conversations:viewAny')) {
        return;                                   // sees the whole account
    }

    // otherwise: only rows where THIS user is in cr.user_ids
    $query->whereRaw(
        'JSON_OVERLAPS(cr.user_ids, JSON_ARRAY(?))',
        [(int) auth()->user()->id]
    );
}
```

`cr.user_ids` is a JSON array of the agent ids who handled that conversation
(`ConversationReport.php:26,51` — cast to array).

- With `viewAny` → no restriction.
- Without `viewAny` (i.e. only `view`) → restricted to conversations the user
  joined. If they joined none in the date range, the result is **empty**.

This is the line whose answer flips between web and worker. Everything in §6
is about why `can('conversations:viewAny')` returns a **different boolean** in the
export job than it does on the web — for the **same user**.

---

## 5. User-based *filters* (optional — user-chosen)

Driven by the `user_ids` filter field, dispatched in `applyFilter()`:

```php
// app/Models/Traits/Filters/ConversationReports/ConversationReportFilterTrait.php:71
case 'user_ids':
    $this->applyUserIdsFilter($query, $value);
    break;
```

```php
// app/Models/Traits/Filters/ConversationReports/Criteria/UserFilterTrait.php:9
private function applyUserIdsFilter(Builder $query, $value): void
{
    $values   = collect((array) $value)->map(fn($id) => (int) $id);
    $hasNoUser = $values->contains(0);     // 0 == "unassigned"
    $userIds   = $values->filter();

    if ($userIds->isEmpty() && !$hasNoUser) return;

    $query->where(function ($q) use ($userIds, $hasNoUser) {
        if ($userIds->isNotEmpty()) {
            $placeholders = implode(',', array_fill(0, count($userIds), '?'));
            $q->whereRaw("JSON_OVERLAPS(cr.user_ids, JSON_ARRAY($placeholders))", $userIds->values()->all());
        }
        if ($hasNoUser) {
            $q->{$userIds->isNotEmpty() ? 'orWhereRaw' : 'whereRaw'}(
                "(cr.user_ids IS NULL OR JSON_LENGTH(cr.user_ids) = 0)"
            );
        }
    });
}
```

- Pass agent ids → keep conversations any of them handled (`JSON_OVERLAPS`).
- Pass `0` → include **unassigned** conversations (empty `user_ids`).
- The permission scope (§4) is applied **first and independently**, so a
  `view`-only user filtering by another agent's id still only sees their own
  conversations (the two `WHERE`s AND together). The user filter cannot widen
  what permissions allow.

### Side note: the agent dropdown list

`getUserIdsFromParams()` (`ConversationReport.php:216`) pulls the distinct agent ids
present in the date range to resolve names for display. **It is only account- and
date-scoped — NOT permission-scoped:**

```php
ConversationReport::query()->select('cr.user_ids')
    ->from('conversation_reports as cr')
    ->where('cr.account_id', auth()->user()->account_id)
    ->whereBetween('cr.opened_at', [$from, $to])
    ->pluck('user_ids')->filter()->flatten(1)->unique()->values()->toArray();
```

This is why the production trace shows this query returning rows (data exists for
the day) while the permission-scoped `count(*)` returns 0 — the data is there, the
viewer just isn't allowed to see it.

---

## 6. How permissions are actually resolved (and where it breaks)

### 6.1 The machinery

This app uses `spatie/laravel-permission`, but roles are **multi-tenant**: each role
row carries `account_id` and/or `organization_id` (migrations
`2020_06_09_102002_add_account_id_to_roles_table.php`,
`2025_10_18_000001_add_organization_id_to_roles.php`).

To make `can()` only count roles belonging to the user's current account/org, a
**custom registrar** overrides the cache key and the role eager-load:

```php
// app/Auth/PermissionRegistrar.php:10
protected function initializeCache()
{
    $cacheKeySuffix = '';
    if (auth()->guard('api')->check()) {
        $user = auth()->guard('api')->user();
        $cacheKeySuffix = '.' . $user->account_id;
        if ($user->organization_id) {
            $cacheKeySuffix .= '.' . $user->organization_id;
        }
    }
    self::$cacheKey = config('permission.cache.key') . $cacheKeySuffix;  // ← STATIC
    ...
}

// :36 getPermissions() loads ONCE, then caches in memory + store:
if ($this->permissions === null) {
    $this->permissions = $this->cache->remember(self::$cacheKey, ..., function () {
        return Permission::with(['roles' => function ($q) {
            return $q->when(auth()->guard('api')->check(), function ($q) {
                $user = auth()->user();
                $q->where('account_id', $user->account_id)
                  ->when($user->organization_id, fn($q) =>
                       $q->orWhere('organization_id', $user->organization_id));
            });
        }])->get();
    });
}
```

Two design facts that combine into the bug:

1. **`self::$cacheKey` is `static` and is computed only once** — in the base
   registrar's constructor (`vendor/spatie/laravel-permission/src/PermissionRegistrar.php:49`
   calls `initializeCache()`). The key is frozen to whatever auth state existed
   **at the moment the registrar singleton was first constructed.**
2. **`$this->permissions` is loaded once per registrar instance and never reset**
   between uses — the only `forgetCachedPermissions()` call in app code is in
   `app/Http/Controllers/OrganizationController.php:48` (HTTP only, on account switch).

The registrar is a **singleton** (`app/Providers/PermissionServiceProvider.php:42`).

### 6.2 Why **viewing on the web is correct**

Every HTTP request is a fresh PHP process:

1. Auth middleware authenticates the user **first**.
2. The registrar is then constructed **after** auth → `initializeCache()` sees the
   logged-in user → key = `spatie.permission.cache.<account>.<org>`.
3. Roles are scoped to that account/org → the user's roles match → `viewAny`
   resolves correctly.
4. Process dies at end of request. One process = one user. **Always consistent.**

### 6.3 Why **export in the worker is wrong**

The export runs in a **long-lived queue worker** that handles many jobs for many
different users:

1. The worker boots and the registrar singleton is constructed **before any user is
   authenticated** → `auth('api')->check()` is `false` → the key is frozen to the
   **bare** `spatie.permission.cache` (no account/org suffix).
   *(The production trace confirms this: `cache:hit spatie.permission.cache` with no suffix.)*
2. A job calls `auth()->setUser($user)` (`ConversationReportsExport.php:52`) — but
   `initializeCache()` is **never run again**, so the key is never recomputed.
3. The **first** job to trigger a permission check loads `$this->permissions` and
   caches it under the bare key — scoped to **that first job's user**.
4. **Every subsequent job in the same worker** finds `$this->permissions !== null`
   and **reuses the first user's permission snapshot**, regardless of who the current
   job's user is. Their own roles are never evaluated.

**Consequences — this hits everyone, not just org admins:**

- A user who genuinely **has** `conversations:viewAny` can have it resolve to
  **false** during export, because the worker is holding a different user's (or an
  empty/unscoped) permission snapshot → their export silently falls back to
  "only my own conversations" (§4) → partial or empty result.
- An **organisation-admin** whose `viewAny` comes via an `organization_id`-scoped
  role is doubly affected: the bare key never even applies the org scoping.
- It is **non-deterministic**: the same user's export can succeed or fail depending
  on which user the worker processed just before, and whether the worker was freshly
  booted.

This is why **"works on the web, broken (or inconsistent) on export"** for the
*same user with the same permissions* — the web evaluates permissions per-request
and correctly; the worker evaluates them once and **leaks that result across users
and jobs**.

### 6.4 Mapping to the production trace

```
select cr.user_ids from conversation_reports ...      → getUserIdsFromParams: rows EXIST for the day
cache:hit spatie.permission.cache                      → bare key (no .account.org) — wrong/leaked scope
select count(*) ... from conversation_reports ...      → permission-scoped count = 0
Exception: "Conversation report list is empty."        → ConversationReportsExport.php:59
```

The error itself is just a guard (`ConversationReportsExport.php:58`) that throws
when the (wrongly) scoped query returns zero rows — it is a *symptom*, not the bug.

---

## 7. Summary of what's wrong

| # | Problem | Location | Effect |
|---|---|---|---|
| 1 | Permission cache key is **static** and computed once at construction | `app/Auth/PermissionRegistrar.php:10` + base ctor | In a worker it freezes to the bare, unscoped key |
| 2 | Registrar is a **singleton**; `$this->permissions` is never reset between jobs | `app/Auth/PermissionRegistrar.php:38`, `PermissionServiceProvider.php:42` | First user's permission snapshot leaks to all later jobs/users in the worker |
| 3 | The job **re-evaluates authorization** in a context that can't resolve it | `ConversationReport.php:475` called from `ConversationReportsExport::handle()` | `viewAny` flips to false → falls back to "own conversations only" |
| 4 | Net result | — | Export ≠ view for the same user; intermittent; empty/partial exports |

The org-admin "missing grant" theory was a red herring — the org admin *does* get
`viewAny` on the web. The real defect is the **shared, frozen permission state in the
queue worker**.

---

## 8. Recommended fix

**Resolve authorization at the request boundary (HTTP, where it's correct) and pass
the decision into the job as plain data. The worker should not re-run `can()`.**

1. In `ConversationReportController` (`index` and `export`), compute the booleans
   once from the authenticated request user:
   ```php
   $params = $request->validated();
   $params['_can_view_any']      = $request->user()->can('conversations:viewAny');
   $params['_actor_id']          = $request->user()->id;
   $params['_can_view_reviews']  = $request->user()->can('conversation:review:view');
   ```
2. `applyPermissionScope(Builder $query, array $params)` reads `$params['_can_view_any']`
   / `$params['_actor_id']` instead of `auth()`.
3. `ConversationReportsExport` uses the passed `_can_view_reviews` instead of
   `$this->user->can(...)` (`ConversationReportsExport.php:115,177` have the same latent bug).

Why this is the right shape:
- Authorization is decided **once, at the boundary** where the HTTP auth context is
  valid; the job **trusts** it (standard Laravel practice — queued jobs have no
  request/auth context).
- No global mutable state (`auth()`, static cache key, singleton snapshot) drives
  correctness → deterministic in HTTP, CLI, and workers alike.
- The registrar bug stops being load-bearing for exports. (Still worth fixing the
  registrar separately, but it no longer corrupts report results.)

*Patching the cache key (recomputing the suffix per call) is NOT recommended: it
keeps the fragile design — global auth state driving scoping, per-tenant entries in
a shared cache, and singleton state shared across jobs — and only narrows the
window, it doesn't close it.*

---

# Part II — Extensive research: the three visibility systems and their logical incompatibilities

The export bug above is one symptom. The deeper problem is that **"who can see a
conversation" is decided by three separate, independently-evolved mechanisms that
were never reconciled.** Each surface of the app (inbox, dashboard, detailed reports,
export) wires up a *different subset* of them. That is the source of nearly every
"why can X see it here but not there" report.

## 9. The three independent visibility mechanisms

| # | Mechanism | Backing data | Tied to RBAC? | Scope semantics |
|---|---|---|---|---|
| **A** | **RBAC permission** | spatie roles, `account_id`/`organization_id`-scoped (custom `PermissionRegistrar`) | yes (it *is* RBAC) | `conversations:viewAny` ⇒ whole account; else participant-only |
| **B** | **Department membership** | `queue_has_users` pivot (`User::departments()`) | **no** | member ⇒ that department's conversations |
| **C** | **Channel skills** | `user_skills` (`skill_type IN (department, channel, channel_line)`) | **no** | no channel skills ⇒ everything; else only skilled channel-lines |

B and C are **orthogonal to the permission system** — they are raw query scopes
applied (or not) per controller. There is no policy, no permission, and no central
resolver that combines all three. Each read surface picks its own combination:

| Surface | A (RBAC) | B (department) | C (skills/channel-line) | Code |
|---|---|---|---|---|
| Live conversation list (inbox) | ✅ `viewAny`/`scopeMine` | ✅ `scopeMyDepartment` | ✅ `getAccessibleChannelLineIds()` | `ConversationsController::index` :140-141; `Conversation.php:332,341`; `InboxController.php:76` |
| Dashboard summary | ✅ | ❌ | ✅ | `DashboardController.php:61-79` |
| Dashboard "latest closed" (uses `ConversationReport`) | ✅ | ❌ | ✅ | `DashboardController.php:71-79` |
| **Detailed Conversation Reports (view + export)** | ✅ `applyPermissionScope` | ❌ **none** | ❌ **none** | `ConversationReport.php:473`; `FiltersConversations.php` |
| Message search | ✅ | ❌ | ❌ | `MessageSearchService.php:19-27` |
| Live conversation policy (`view`) | ✅ + `users` pivot | ❌ | ❌ | `ConversationPolicy.php:25` |

**The single most consequential row:** the detailed Conversation Reports surface —
the one the export uses — applies **only** mechanism A. Department membership and
skills are silently dropped. So the report's notion of "what I'm allowed to see" is
*narrower* than the inbox's (no department branch) on one axis and *wider* on another
(no channel-skill restriction). They cannot agree by construction.

## 10. Department membership — how it works, and the report gap

**Model.** "Department" == "queue". The model is `App\Models\Asterisk\Queue`; the PK
is a UUID in the `name` column, with a human `display_name`. Reports join
`queues as q ON q.name = cr.department_id` (`ConversationReport.php:407`).

**Membership.** `queue_has_users` pivot (`queue_name`, `user_id`, + `auto_assign`).
- `User::departments()` → `belongsToMany(Queue, 'queue_has_users')` (`User.php:174`).
- `Queue::users()` (`Queue.php:113`).
- **There are NO roles within a department** — the pivot has only `auto_assign`; no
  `is_supervisor` / `is_manager` / `role` column exists anywhere, and there is **no
  `supervisor`/`manager` permission** in the system. Membership is flat.

**How a conversation gets a department.** `conversations.department_id` (UUID FK →
`queues.name`, `ON DELETE SET NULL`) is set at creation from the widget's department
(`Widgets/ConversationsController.php:53`) and on transfer
(`ConversationsController::transferToDepartment` :290). The report row copies it
through.

**The gap (confirmed asymmetry):**

```php
// LIVE conversations — department membership DOES grant visibility
// app/Models/Conversation.php:341
public function scopeMyDepartment(Builder $query): Builder
{
    if (!auth()->check() || auth()->user()->can('conversations:viewAny')) return $query;
    return $query->whereIn('conversations.department_id',
        auth()->user()->departments->pluck('name'));
}
```

```php
// REPORTS — no department branch at all
// app/Models/ConversationReport.php:473
protected function applyPermissionScope(Builder $query): void
{
    if (auth()->user()->can('conversations:viewAny')) return;
    $query->whereRaw('JSON_OVERLAPS(cr.user_ids, JSON_ARRAY(?))', [(int) auth()->user()->id]);
}
```

**Concrete incompatibility:** a department member without `viewAny` sees their
department's **live** conversations in the inbox, but opening **Reports** for the
exact same conversations returns nothing unless they personally appear in
`cr.user_ids`. `department_ids` exists on the report side only as an *optional
client filter* (`allowedFilterFields`, mapped to `cr.department_id`) — it ANDs on top
of the participant scope, so it can never *widen* what a non-`viewAny` user sees.

## 11. Skills — how they work, and the inconsistent report application

**Model.** A "skill" is not a catalog entity; it's a row in `user_skills`
(`skill_type ∈ {department, channel, channel_line}`, `skill_id` = the target's id,
`extra_attributes.auto_assign`). Created `2026_01_10`. No `account_id`/org column —
skills hang off the user. `UserSkill` constants at `UserSkill.php:13-15`.

**Two jobs, both real:**
1. **Routing.** `OperatorService.php:110-156` uses `User::withChannelAccess()`
   (`User.php:331`): an operator with **no** channel skills matches everything;
   otherwise only conversations on a channel-line/channel they're skilled in (and,
   for auto-assign, with `auto_assign=true`). Department routing uses the separate
   `queue_has_users` table.
2. **Visibility.** `User::getAccessibleChannelLineIds()` (`User.php:281-301`) returns
   `null` ("see everything") when the user has no channel skills, else an explicit
   channel-line id list. This filters the **inbox** (`InboxController.php:76`),
   **visitor list** (`InboxVisitorController.php:51`), **dashboard**
   (`DashboardController.php:67`), and the **dashboard-summary report**
   (`GetConversationsReport.php:50,68-88`).

**The inconsistency (confirmed):** the **detailed** Conversation Reports listing
(`GetConversationReports` → `FiltersConversations` and `ConversationReport::report()`)
applies **no skill / channel-line filtering at all**. `FiltersConversations.php`
filters only by date range and type; `ConversationReport` exposes `channel_line_id`
as a column (`:41`) but never scopes by it. So:

- The **dashboard's** "latest closed conversations" widget is channel-skill scoped.
- The **Reports page** showing the *same underlying rows* is **not**.

A channel-skilled agent therefore sees a *different population* of historical
conversations on the dashboard vs. the reports page — and the export inherits the
reports-page behavior (no skill scope). Also note skill filtering is applied
**unconditionally, on top of permissions** (even for `viewAny` users on the
dashboard), whereas the report permission scope is an either/or — another layering
mismatch.

Skills have **zero** connection to the spatie permission/role system; `syncRoles()`
and `syncSkills()` are independent (`UsersController.php:111-112`). The account-level
`user_skills` boolean only toggles realtime **broadcast** channel granularity
(`BroadcastsOnConversationChannels.php:23,52`), not DB-level filtering.

## 12. What "participation" (`user_ids`) actually means — and who it wrongly excludes

The report permission scope (mechanism A, non-`viewAny` branch) hinges entirely on
`cr.user_ids`. That column is frozen at report-generation time from **`join`
conversation-events only**:

```php
// app/Actions/Reports/GenerateConversationReport.php:272
->selectRaw('if (count(user_id) > 0, json_arrayagg(user_id), null) as user_ids')
// user_id sourced from join/leave event pairs — HasPairsQuery.php:45
```

Event→type mapping (`ConversationEventService.php:31-48`): `UserAttachedToConversation`
and `ChatBotAttachedToConversation` ⇒ `join`; `ConversationHeld` ⇒ `hold` (separate
`hold_user_id`); bots tracked in `chat_bot_durations`. Consequences:

| Involved party | In `user_ids`? | Result for non-`viewAny` viewer |
|---|---|---|
| Agent who messaged / was assigned | ✅ | can see report |
| Transferred-**from** and transferred-**to** agents | ✅ (both keep join/leave pairs) | both can see — *not* a gap |
| **Supervisor who only put it on hold** (barge/listen) | ❌ (lands in `hold_user_id`) | **cannot see the report** |
| **Intended recipient of an unanswered/abandoned conv** | ❌ (`user_ids` is `NULL`) | **cannot see** — and `JSON_OVERLAPS(NULL, …)` is NULL/false, so non-`viewAny` users see **none** of these abandoned rows |
| Bot / AI assistant | ❌ | n/a (not a human viewer) |

Two distinct ownership definitions also disagree across surfaces: the **live**
`ConversationPolicy::view` (`:25`) uses `$conversation->users->contains($user)` (the
**current** pivot membership), while the **report** uses the **frozen** `user_ids`
JSON. A user detached from the live conversation can still be in the historical
`user_ids` (or vice-versa).

## 13. Consolidated list of logical incompatibilities

1. **Department membership grants live visibility but not report visibility.**
   `scopeMyDepartment` exists for `Conversation`; there is no equivalent on
   `ConversationReport`. (§10)
2. **Channel skills scope the inbox + dashboard + dashboard-summary report, but not
   the detailed reports / export.** Same data, different population depending on the
   page. (§11)
3. **Three visibility mechanisms, no central resolver.** Each surface hand-picks a
   subset (§9 table), guaranteeing they diverge. RBAC is policy-driven; department
   and skills are ad-hoc query scopes with no policy/permission backing.
4. **`conversations:view` gates the endpoint but never widens row scope.** A `view`
   (non-`viewAny`) user is silently reduced to participant-only — `view` and `viewAny`
   are not two points on a spectrum, they're "reach the page" vs. "see everything."
   (§2, `ConversationReportRequest.php:21`)
5. **Hold-only supervisors and abandoned-conversation recipients are excluded** from
   their own reports because `user_ids` is built from `join` events only and is `NULL`
   for unanswered conversations. (§12)
6. **Live vs. report ownership use different definitions** (`users` pivot vs. frozen
   `user_ids` JSON). (§12)
7. **`getUserIdsFromParams()` is account+date scoped but NOT permission/department/
   skill scoped** (`ConversationReport.php:216`). The agent-name lookup / dropdown it
   feeds therefore exposes the **full set of account agent ids and names** for the
   period to a viewer who is otherwise restricted to their own conversations — a minor
   information leak and another inconsistency with the row-level scope.
8. **The own-scoping is currently latent.** `Role::createForAccount()` grants the
   default `operator` role *all* permissions including `conversations:viewAny` (the
   exclusion at `Role.php:83` is commented out). So in the default config nobody is
   participant-scoped and incompatibilities #1/#4/#5 don't bite — until an admin
   creates a **custom role** with `view` but not `viewAny`, at which point all of them
   activate at once. The behavior is a cliff, not a gradient.
9. **The org-admin role has no permissions granted by any code path** (no org
   equivalent of `createForAccount`); org-scoped admin roles are created out-of-band.
   (See Part I context.)
10. **The export worker resolves permissions from leaked/frozen singleton state** (Part
    I §6), so even where mechanism A *would* be correct, the export can disagree with
    the view for the same user. This sits on top of #1–#9 and makes them
    non-deterministic for exports specifically.

## 14. How department/skills *should* interact — and a recommendation

The product clearly intends layered visibility: **RBAC** (can you reach reports at
all, and do you get the whole account) → **department membership** (your team's
conversations) → **channel skills** (your channels). Today only the inbox composes
all three; reports compose one.

To make reports consistent with the inbox, the report scope needs to mirror the live
scope — conceptually:

```
visible(report) =
    canViewAny                                   // mechanism A override
 OR actorId ∈ report.user_ids                    // participation (existing)
 OR report.department_id ∈ actor.departmentNames // mechanism B (MISSING on reports)
AND ( actor has no channel skills                // mechanism C (MISSING on reports)
      OR report.channel_line_id ∈ actor.accessibleChannelLineIds )
```

Recommended direction (consistent with the Part I fix):

1. **Centralize the rule.** Build one resolver (e.g. a `ConversationVisibility` value
   object / query scope) that takes an explicit *actor context* — `canViewAny`,
   `actorId`, `departmentNames`, `accessibleChannelLineIds` — and is used by **both**
   `Conversation` and `ConversationReport`. This removes the per-surface drift (#1, #2,
   #3) by construction.
2. **Resolve the actor context at the HTTP boundary and pass it into the job as data**
   (per Part I §8). The worker never calls `auth()`/`can()` and never reads department
   or skill state through global helpers — it just applies the precomputed context.
   This simultaneously fixes the export determinism bug (#10) and lets reports honor
   department + skill scoping the same way the inbox does.
3. **Decide the participation definition deliberately** (#5, #6): should hold-only
   supervisors and intended-recipients-of-abandoned-conversations count? If yes,
   include `hold_user_id` / queued-recipient ids in the visibility test (not
   necessarily in `user_ids`, but in the scope), and handle `user_ids IS NULL`
   explicitly so abandoned rows aren't silently invisible.
4. **Permission-scope `getUserIdsFromParams()`** (#7) so the agent dropdown reflects
   what the viewer may actually see.
5. **Resolve the latent cliff** (#8): decide whether default operators should be
   own-scoped (restore `Role.php:83`) or genuinely see-all, so the participant-scope
   code path is either truly dead or actually exercised and tested.

Without a central resolver, every new report surface will re-introduce a different
subset of these three systems and the incompatibilities will keep multiplying.
