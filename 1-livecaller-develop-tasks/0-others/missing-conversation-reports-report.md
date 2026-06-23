# Missing `conversation_reports` — Root Cause Analysis & Fixes

## Summary

Daily, some `conversations` exist with **no matching `conversation_reports` row**. Investigation of
production data (the missing-report conversations + the `failed_jobs` table) revealed **three
independent root causes**, all ending the same way: the report is never written.

A `conversation_report` is **not** created when a conversation is created. The chain is:

1. `Conversation::close()` ([app/Models/Conversation.php:361](app/Models/Conversation.php)) dispatches
   `ConversationClosed` — **only if `$fireEvent === true`** (the default).
2. `ConversationClosed` drives two things:
   - **Synchronously:** `ConversationEventService::process` writes the `close` row into
     `conversation_events`.
   - **Queued (`ShouldQueue`):** `GenerateReportForClosedConversation` →
     `Conversation::generateReport()` → `GenerateConversationReport`.
3. `GenerateConversationReport` builds the report entirely from `conversation_events` and requires
   **both** an `open` row and a non-null `close`
   (`->where('events.event','open')->whereNotNull('events.close')`,
   [app/Actions/Reports/GenerateConversationReport.php:237-238](app/Actions/Reports/GenerateConversationReport.php)).

So a report only appears when a conversation (a) fired `ConversationClosed`, (b) has both `open` and
`close` rows, and (c) whose queued job actually ran without throwing. Each cause below breaks one of
these.

---

## Cause #1 — Outgoing social-hub conversations closed with `fireEvent = false`

**Where:** [app/Actions/SocialHub/GetConversation.php:112](app/Actions/SocialHub/GetConversation.php)

```php
} elseif ($conversation->wasRecentlyCreated) {
    $conversation->close($conversation->account, false); // fireEvent = false
}
```

**What happens:** For outgoing-initiated social-hub threads (messenger, etc.), a freshly created
conversation is closed with `fireEvent = false`. `ConversationClosed` is never dispatched, so
`ConversationEventService` never writes a `close` event. The report query requires
`whereNotNull('events.close')`, so these can **never** produce a report. The
`closed_by_member_type = account` column on the row is written directly by `close()`'s `forceFill`,
which is why the conversation *looks* closed but has no close **event**.

**Pattern in the data:** ~42% of the missing reports. All `conversation_type_id = 3` (hub), closed by
`account`, many with `created_at == closed_at` (instant close). Accounts: 5030, 4831, 2543, 4646,
4607, 2902 (messenger/social-hub).

**Why backfill does NOT fix these:** there are no `open`/`close` events to build a report from.

---

## Cause #2 — Negative `in_queue_duration` overflows an UNSIGNED column

**Where:** insert at [app/Actions/Reports/GenerateConversationReport.php:47](app/Actions/Reports/GenerateConversationReport.php),
value computed in `getReport()`.

**Confirmed by `failed_jobs`:**

```
SQLSTATE[22003]: Numeric value out of range: 1264 Out of range value for column 'in_queue_duration' at row 1
... insert into `conversation_reports` (... `in_queue_duration` ...) values (..., -2, ...)
```

The column is `unsignedInteger`
([database/migrations/2020_10_12_104418_create_conversation_reports_table.php:28](database/migrations/2020_10_12_104418_create_conversation_reports_table.php)),
and the job tries to insert **`-2`**.

**Root cause — AI-handoff timing race.** From the failed insert:
- `opened_at` = 15:01:52, `answered_at` = **15:04:07**, `closed_at` = 15:28:49
- `automated_transferred_at` (from `extra_attributes`) = **15:04:09**

The `in_queue_duration` formula for the transferred branch:

```sql
WHEN automated_transferred_at IS NOT NULL
  THEN TIMESTAMPDIFF(SECOND, automated_transferred_at, COALESCE(answered_at, p2.close))
```

= `TIMESTAMPDIFF(SECOND, '15:04:09', '15:04:07')` = **−2**.

The operator was attached (`answered_at`) **2 seconds before** the bot's `automated_transferred_at`
was stamped — a normal race when a human picks up right as the AI hands off. The negative value goes
into an UNSIGNED column → `1264 Out of range` → the queued job throws → no report.

**Pattern in the data:** website conversations (`conversation_type_id = 1`) with an AI assistant +
`automated_transferred_at`. Clusters on account **24** (widget `e6b7026a`, which runs the AI assistant
+ auto-transfer) and recurs daily.

**Why backfill DOES fix these:** the `force` path strips `STRICT_TRANS_TABLES` from the session
`sql_mode` before its `insertUsing`, so MySQL silently clamps `-2` → `0` instead of erroring. The live
(strict-mode) job has no such protection.

---

## Cause #3 — Stale polymorphic `closedBy` relation breaks job deserialization on retry

**Where:** `ConversationClosed` event ([app/Events/Conversation/ConversationClosed.php](app/Events/Conversation/ConversationClosed.php)),
`closedBy` is a `MorphTo` ([app/Models/Conversation.php:154](app/Models/Conversation.php)).

**Confirmed by `failed_jobs`:**

```
RelationNotFoundException: Call to undefined relationship [permissions] on model [App\Models\Visitor].
... App\Events\Conversation\ConversationClosed->restoreModel() ... unserialize()
```

This crashes during job **deserialization**, before `handle()` ever runs.

**Root cause.** The serialized event captured the conversation with these relations:
`closedBy`, **`closedBy.permissions`**, **`closedBy.roles`**, `coBrowseSessions`, `visitors`. The
nested `permissions`/`roles` relations exist only on `User` — but `closedBy` is polymorphic.

Sequence:
1. A logged-in operator (`User`, with `permissions`/`roles` already loaded from auth) closes the
   conversation → `close()` calls `$this->closedBy()->associate($closedBy)`
   ([Conversation.php:369](app/Models/Conversation.php)), so the conversation's `closedBy` relation
   holds that `User` *with its nested relations*. `SerializesModels` records the relation paths
   `closedBy.permissions`, `closedBy.roles`.
2. The job is queued. Before it runs (note `attempts: 2` — a **retry**), the conversation is
   **reopened and re-closed by a visitor** (the hub reopen path, `shouldReopen()` for type `hub`).
   Now `closed_by` points to a `Visitor`.
3. On restore, Laravel re-fetches the conversation and replays
   `load(['closedBy', 'closedBy.permissions', 'closedBy.roles', ...])`. `closedBy` now resolves to a
   `Visitor`, which has no `permissions` relation → `RelationNotFoundException` → the job cannot
   deserialize → no report.

**Pattern:** any conversation re-closed by a visitor between dispatch and processing — common on hub
conversations that reopen.

**Why backfill DOES fix these:** the command builds directly from `conversation_events` and never
deserializes a queued event, so it sidesteps this entirely.

---

## Cause comparison

| # | Trigger | Failure | Recovered by backfill? |
|---|---------|---------|------------------------|
| 1 | Outgoing social-hub closed `fireEvent = false` | No `close` event → report query never matches | ❌ no events to build from |
| 2 | AI handoff: `answered_at` precedes `automated_transferred_at` → negative `in_queue_duration` | `1264 Out of range` on `unsignedInteger` | ✅ command runs in non-strict `sql_mode` |
| 3 | Conversation re-closed by visitor before queued job restores | `RelationNotFoundException [permissions] on Visitor` at deserialization | ✅ command doesn't serialize events |

---

## Fixes

### Fix #2 — clamp durations in `getReport()`

In `app/Actions/Reports/GenerateConversationReport.php`, wrap the duration expressions in
`GREATEST(0, …)` so the stored value is semantically correct (`0`, not a MySQL-clamped artifact) and
the live path stops crashing:

```sql
GREATEST(0, CASE
  WHEN has_automated_handler = 1 AND automated_transferred_at IS NULL THEN 0
  WHEN automated_transferred_at IS NOT NULL
    THEN TIMESTAMPDIFF(SECOND, automated_transferred_at, COALESCE(answered_at, p2.close))
  ELSE TIMESTAMPDIFF(SECOND, p2.open, COALESCE(answered_at, p2.close))
END) as in_queue_duration
```

Apply the same `GREATEST(0, …)` wrap to `answered_duration`. `total_duration` is always `open → close`
(positive), so it is safe. Because both the live and `force` paths share `getReport()`, this one
change fixes both.

### Fix #3 — stop serializing User-only nested relations on the polymorphic `closedBy`

In `app/Events/Conversation/ConversationClosed.php`, clear relations on the models in the constructor.
Broadcasting already re-loads what it needs via `loadMissing()` in `broadcastWith()`, so this is safe:

```php
public function __construct(Conversation $conversation, Model $closedBy = null)
{
    $this->conversation = $conversation->withoutRelations();
    $this->closedBy = $closedBy?->withoutRelations();
}
```

This prevents the `closedBy.permissions` / `closedBy.roles` paths from ever being recorded, so restore
can never trip over a now-`Visitor` closer.

### Fix #1 — generate the report in the outgoing-hub branch

In `app/Actions/SocialHub/GetConversation.php`, the outgoing branch should either fire the event or
call `generateReport()` directly (mirroring the compensating email path in
[app/Actions/Conversation/SendMessage.php:297-303](app/Actions/Conversation/SendMessage.php)), so the
`open`/`close` events and report are produced for these threads.

### Fix #4 — schedule the backfill as a safety net

`lc:conversations:generate-reports`
([app/Console/Commands/GenerateConversationReports.php](app/Console/Commands/GenerateConversationReports.php))
regenerates reports for all `Conversation::doesntHave('reports')`. It is **not currently in the
scheduler**, so nothing self-heals today. Add it (e.g. nightly) in `app/Console/Kernel.php`. It already
recovers causes #2 and #3 automatically.

```php
// app/Console/Kernel.php — schedule()
$schedule->command('lc:conversations:generate-reports')->dailyAt('03:00');
```

---

## How to confirm on production

```sql
-- 1. Closed conversations with no report (the real problem set)
SELECT c.id, c.conversation_type_id, c.department_id, c.closed_at,
       SUM(e.event = 'open')  AS has_open,
       SUM(e.event = 'close') AS has_close,
       JSON_EXTRACT(c.extra_attributes, '$.social_hub.provider.name')   AS hub,
       JSON_EXTRACT(c.extra_attributes, '$.automated_transferred_at')   AS transferred_at
FROM conversations c
LEFT JOIN conversation_reports r ON r.conversation_id = c.id
LEFT JOIN conversation_events  e ON e.conversation_id = c.id
WHERE c.closed_at IS NOT NULL AND r.id IS NULL
GROUP BY c.id
ORDER BY c.closed_at DESC
LIMIT 200;
-- has_close = 0  -> Cause #1 (fireEvent=false / missing close event)
-- has_open=1, has_close=1, transferred_at present -> Cause #2 (negative in_queue_duration)

-- 2. The actual exceptions
SELECT failed_at, LEFT(exception, 200) AS ex
FROM failed_jobs
WHERE payload LIKE '%GenerateReportForClosedConversation%'
ORDER BY failed_at DESC
LIMIT 50;
-- '1264 Out of range value for column in_queue_duration' -> Cause #2
-- 'RelationNotFoundException ... [permissions] on model [App\Models\Visitor]' -> Cause #3
```

## Recovering the already-missing reports

After deploying the fixes, backfill (runs the `force` path, which survives all crash modes that have
events):

```bash
docker exec livecaller-php-api-1 php /var/www/api/current/artisan lc:conversations:generate-reports
```

Note: jobs already in `failed_jobs` will **not** auto-retry; the backfill command is what recovers
them (except Cause #1, which needs the code fix to start producing events).
