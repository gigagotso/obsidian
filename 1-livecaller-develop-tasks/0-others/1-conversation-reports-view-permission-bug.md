# Conversation Reports — `view`-only users can't see their own conversations

**Date:** 2026-06-10 (deep research + live reproduction)
**Area:** API (`www/api`) + Frontend (`www/app`)
**Symptom:** A user with only `conversations:view` (not `conversations:viewAny`) opens Conversation Reports, filters by their own user, and sees nothing — even though their conversations exist.

## TL;DR — the exact mechanism

1. The frontend **menu and page** are gated on `conversations:view` → the page opens fine for view-only users.
2. Every data fetch on that page (initial load AND filter-by-user) calls `GET /conversations/reports`.
3. The backend FormRequest `authorize()` hard-requires `conversations:viewAny` → returns **HTTP 403 before any query runs**.
4. The frontend catches the error, sets `rows = []`, and renders an **empty table** (plus a generic "You don't have permission" toast from the axios interceptor).
5. Result: the page looks like "my conversations don't show up", when in reality **every request from that page is rejected** — the user filter is irrelevant.

**Permission mismatch in one line:** frontend gates the page on `conversations:view`, backend gates the data on `conversations:viewAny`.

## Live reproduction (verified in Docker, 2026-06-10)

```php
$user = User::find(207);                      // has conversations:view, NOT viewAny
$user->can('conversations:view');             // true
$user->can('conversations:viewAny');          // false

auth()->setUser($user);
(new ConversationReportRequest())->authorize(); // false  → 403

// Their data DOES exist — the report table contains them:
DB::table('conversation_reports')
    ->where('account_id', $user->account_id)
    ->whereRaw('JSON_CONTAINS(user_ids, ?)', [(string) $user->id])
    ->count();                                 // 2035 rows
```

**89 users** in the current DB have `conversations:view` without `viewAny` — all of them hit this.

Permission resolution chain (so there's no doubt where `can()` lands): the app uses **spatie/laravel-permission** (`HasRoles` on `User`); both `conversations:view` and `conversations:viewAny` exist as separate permissions in the DB. No wildcards, no `Gate::before` admin override, and the custom `App\Auth\Gate` only adds `Policy@method` syntax. So `can('conversations:viewAny')` is a strict DB permission check — view-only users genuinely fail it.

## How it regressed (timeline)

| When | Commit | What happened |
|---|---|---|
| pre-Nov 2024 | — | Old **Conversation History** page (`GET /conversations/history`) used `GetConversationHistory` → `Conversation::scopeMine()`: `viewAny` → all conversations, `view`-only → conversations you participated in. **Worked correctly.** |
| Nov 12 2024 | `ac19c329` | New Conversation Reports feature added. `authorize()` = "any logged-in user" and the report query has **no own-user scoping** → at this point view-only users could see **all** account conversations (an over-exposure bug in the other direction). |
| Dec 19 2024 | `d4605a3d` ("reafactoring") | The over-exposure was plugged at the gate: `authorize()` changed to `can('conversations:viewAny')`. But instead of scoping the query down for `view`-only users (the `scopeMine()` pattern), it **locked them out entirely**. |
| later | — | Conversation History page marked **deprecated** in the frontend ("Please use Conversation Reports instead") → view-only users lost their only working path to their own past conversations. |

So the bug isn't one bad line in isolation — it's that the `viewAny` gate was added **without porting the `view` → "own conversations only" fallback**, and the working legacy page was deprecated underneath it.

## Why "filter by user" feels like the trigger

It isn't. Selecting yourself in the user filter sends `filters[user_ids][in]=<your id>` to the **same** endpoint that already 403s. The empty result is identical with or without the filter — the filter just makes the absence of *your own* conversations obvious.

## Key files

### API (`www/api`)
| File | Role |
|---|---|
| `app/Http/Requests/Reports/ConversationReportRequest.php:19-22` | The 403 gate — `can('conversations:viewAny')`, no `view` fallback |
| `app/Models/ConversationReport.php` — `scopeReport()` / `applyAccountScope():463` | Report query — scoped to `account_id` only, no own-user restriction |
| `app/Models/Traits/Filters/ConversationReports/Criteria/UserFilterTrait.php` | Applies any client-sent `user_ids` via `JSON_OVERLAPS`, no permission check |
| `app/Models/Conversation.php:332` — `scopeMine()` | The reference pattern the old endpoint used |
| `app/Policies/ConversationPolicy.php` — `view()` | Intended model: `viewAny` → all, `view` → participant only |

### Frontend (`www/app`)
| File | Role |
|---|---|
| `src/modules/layout/AppNavigation/useNavigationMenu.js:119-125` | Menu item gated on `conversations:view` — the mismatch with the backend |
| `src/router/routes.js:454-460` | Route has `requiresAuth` only, no permission meta |
| `src/modules/reports/conversation-reports/useConversationReports.js:187-249` | `fetchReports()` catch → `rows = []` → silent empty table |
| `src/util/axios.js:165-170` + `src/shared/toast/showApiError.js` | Generic 403 toast |

## Fix — implemented 2026-06-11 (uncommitted, on `develop` working tree)

Mirrors the `scopeMine()` model in the reports stack:

1. **`ConversationReportRequest::authorize()`** — now accepts either permission:
   `can('conversations:viewAny') || can('conversations:view')`.
   `ExportConversationReportRequest` extends it, so the export gate is covered too.
2. **`ConversationReport::applyPermissionScope()`** (new, called from `scopeReport()` right after `applyAccountScope()`) — when the user lacks `viewAny`, unconditionally adds
   `JSON_OVERLAPS(cr.user_ids, JSON_ARRAY(auth()->id()))`
   regardless of any client-supplied `user_ids` filter. Restores own conversations AND prevents `view`-only users from reading other agents' conversations via the filter.
3. **Export path** — covered automatically: `ConversationReportsExport` job does `auth()->setUser($this->user)` before calling `ConversationReportService::generate()`, so the same scope applies in the queue worker.

### Verified in Docker (`livecaller-php-api-1`)

| Check | Result |
|---|---|
| `authorize()` for view-only user 207 | `true` (was `false`) |
| Report rows for user 207 | 2035 — exactly their own; 0 foreign rows on page |
| User 207 filtering by user 208 | 0 rows — no cross-agent leak |
| viewAny user (id 56) | 205,571 rows = full account total — unchanged |

No automated tests added — the project's test suite is essentially empty (single Widget feature test, no factories for this area).

### Optional follow-up (frontend)
Surface API 403s on the reports page as a real error state instead of an empty table (`useConversationReports.js` catch block sets `rows = []`).
