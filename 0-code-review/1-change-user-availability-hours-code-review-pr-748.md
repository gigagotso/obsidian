# Code Review — PR [#748](https://github.com/livecaller/api.livecaller.io/pull/748) "Change user availability hours"

**Date:** 2026-06-10
**Base:** `develop` ← **Head:** `change-user-availability-hours`
**Size:** 31 files, +1761/−471
**Verdict:** Request changes — findings 1–4 are blocking; 5–7 will break or misroute production the day this deploys.

## Overview

Large PR: reworks the hourly availability aggregation command to per-hour slots sourced from `user_availability_logs`, replaces the CPH/availability reports with channel-aware versions, adds multi-sheet Excel exports, moves Lambda function names into config, and ships four migrations plus a test seeder. The aggregation rewrite itself is solid — parameterized bulk upsert, a clean hour-window overlap query, and a supporting index. **However, the PR removes authorization on the entire agent-analytics surface, and a migration drops a column that a still-live endpoint queries.** Both must be fixed before merge. The PR body is also empty for a change that moves routes, alters API contracts, and runs destructive migrations — that needs a rollout/rollback note.

## Critical (must fix before merge)

1. **Permission middleware disabled on the whole controller** — `app/Http/Controllers/AgentAnalyticsController.php:44`
   ```php
   //        $this->middleware('permission:analytics:agent:view');
   //        $this->middleware('permission:conversation:review:view')->only(['satisfaction']);
   ```
   Every endpoint here (`satisfaction`, `availability`, `performance`, `response-times`, `timeline`, and all hourly exports) is now reachable by **any** authenticated user in the account, regardless of permissions. `satisfaction` additionally loses its `conversation:review:view` gate. This looks like a debugging leftover; restore both lines.

2. **Self-scoping removed from report user resolution** — `app/Actions/Reports/Traits/ResolvesReportUsers.php:12`. The old code returned only the requesting user's own ID when they lacked `analytics:agent:viewAny`; that check is deleted, and the new `resolveRequestedUsers()` in `app/Actions/Reports/Traits/QueriesUserAvailabilityHourly.php` never had it. A regular agent omitting `users` now receives availability/CPH/conversation data for **every agent in the account**. Combined with finding 1, intra-account authorization on agent analytics is fully gone. (Tenant isolation is still intact — `getUsers()` scopes through `account->users()` and the FormRequests check `account_id` on `users.*` — but the per-permission scoping must come back.)

3. **Migration drops a column a live endpoint still queries** — `database/migrations/2026_05_19_000002_drop_conversations_answered_from_user_availability_hourly_table.php` drops `conversations_answered`, but `app/Actions/Reports/GetUsersHourlyConversationsReport.php:56` still runs `SUM(COALESCE(conversations_answered, 0))`. That action backs `GET analytics/users/conversations-hourly`, which this PR keeps registered (`routes/api.php:384`). The moment the migration runs, every such export fails with "Unknown column". Either port that report to `conversations_by_channel` or don't drop the column yet. Also note the drop **destroys historical data** with no backfill into the new JSON column — `down()` restores the column but not its contents.

4. **Breaking API contract changes, undocumented**
   - Routes moved from `analytics/agent/{availability-hourly,conversations-hourly,cph-hourly}` to `analytics/users/...` — old URLs now 404.
   - `daily-call-summary` is removed outright (commented at `routes/api.php:386`) while the controller method remains.
   - `from`/`to` validation tightened from `date` to `date_format:Y-m-d H:i` (and `after_or_equal` → `after`) — clients sending `2026-01-01` now get a 422.

   These may all be intentional, but with an empty PR body there's no coordination signal for the frontend. Confirm the consumers are updated in lockstep and document it.

## Major (should fix before merge)

5. **Lambda config defaults to the develop function** — `config/aws-lambda.php:14`: `env('AWS_LAMBDA_EXCEL_EXPORT_FUNC', 'livecallerAppExcelExportDevelop')`. The previous hardcoded value everywhere was `livecallerAppExcelExport`. If the env var isn't set in production, **all** Excel exports (tags, shortcuts, history, conversation reports, hourly) silently invoke the develop Lambda — wrong behavior at best, production PII flowing through a dev function at worst. Default to the production name (or no default + fail loudly).

6. **`account_id` migration fails on a populated table** — `database/migrations/2026_05_19_000001_add_account_id_to_user_availability_hourly_table.php` adds `foreignId('account_id')` NOT NULL with `constrained()` and no backfill. On MySQL, existing rows get `0`, and the FK creation then fails. If the table has data anywhere (it does in any env where the command was run), the deploy breaks. Add as nullable → backfill from `users.account_id` → then constrain.

7. **Nothing schedules the aggregation** — `app/Console/Kernel.php:52` still has the schedule commented out (`// ...->hourlyAt(5)`). All reports in this PR read from `user_availability_hourly`, so in any environment where the command isn't run by hand the feature returns zeros. If this is a staged rollout, say so in the PR; otherwise uncomment it.

8. **Latent SQL injection — validation lives only in the caller** — `app/Actions/Reports/GetUsersHourlyConversationsByChannelReport.php:82` interpolates channel names into `selectRaw` (`'$.' . $ch` and alias `` `ch_{$ch}` ``), relying on the comment "validated via Rule::in(), safe to interpolate." That's true **today** only because `app/Http/Requests/Reports/HourlyConversationsByChannelExportRequest.php:17` runs upstream. The Action itself has no `rules()` (the old actions carried `HasAgentReportValidation`), so any future caller — Nova action, console command, lorisleiva route-as-controller — passes raw input straight into SQL. Sanitize at the point of use (e.g., `preg_match('/^[a-z_]+$/', $ch)` allowlist inside `queryConversationsData`, or intersect with `Channel::pluck('name')` there) rather than trusting every caller.

9. **Seeder is dangerous if run against production** — `database/seeds/AvailabilityReportTestSeeder.php` creates 254 users with the known password `password` on a real account, force-enables the `hourly_export` feature flag, hardcodes environment-specific PBX channel-line IDs (`'960144646930042881'`, … — FK violations or cross-data pollution elsewhere), and inserts ~1.3M rows. Guard it: `if (app()->environment('production')) { abort... }`, and consider random passwords.

## Minor

10. **Signature parsing quirk** — `app/Console/Commands/AggregateHourlyAvailability.php:25`: `{--chunk=100: Number of users per chunk.}` is missing the space before `:`, so Laravel treats the entire string `"100: Number of users per chunk."` as the default value. It only works because of the `(int)` cast; `--help` output is wrong. Use `{--chunk=100 : Number of users per chunk.}`.
11. **Weak option validation** — `--hour=abc` passes (`(int)'abc'` = 0) and silently aggregates hour 0; passing `--date` together with `--from/--to` silently ignores the range. Validate `ctype_digit` and reject conflicting options.
12. **`$fillable` is wrong** — `app/Models/UserAvailabilityHourly.php:14` lists `channel_id` (no such column) and omits `account_id` (added by this PR). Harmless today because the command uses raw SQL, but the first mass-assignment will silently drop `account_id`.
13. **Stale-row staleness on re-aggregation** — `processChunk` skips users with zero activity and the upsert only touches computed rows, so re-running an hour after a data correction leaves previously-written rows at their old values. Worth a `DELETE` for the slot scope (or zero-fill) if backfills are expected to be authoritative.
14. **Misleading class name** — `GetUsersHourlyAvailabilityByChannelReport` has no channel dimension at all (it returns `minutes_online` only). The name will confuse the next reader; `GetUsersHourlyAvailabilityReport` was accurate.
15. **Contract drift between sibling endpoints** — `HourlyConversationsExportRequest` still uses `date`/`after_or_equal` and duplicates `authorize()` instead of extending the new `AbstractHourlyExportRequest`, while its three siblings use `Y-m-d H:i`/`after`. Same UI screen, two shapes.

## Nits

- Commented-out code should be deleted, not kept (`daily-call-summary` route, old schedule line) — unless it's the restore-me marker for the Critical items above.
- `(float)` casts on conversation counts in `app/Jobs/HourlyConversationsByChannelExport.php` — counts are ints.
- The docblock additions (`@return bool` over `authorize(): bool`) are exactly the type noise the repo's `.cursor/rules/laravel.mdc` convention says to avoid.

## What's good

- The aggregation rewrite is a real improvement: the old 70-line window-function SQL with hand-ordered bindings is replaced by a readable hour-overlap query against `AvailabilityLog`, with the supporting composite index added in the same PR, and a fully parameterized bulk upsert (no injection surface there — column names hardcoded, all values bound).
- `sanitizeForExcel` is consistently applied to user names in all the new sheet builders — formula-injection in exports is covered.
- Per-hour idempotent slots with `min(x, 3600)` capping and the `--date/--hour/--from/--to` backfill ergonomics are well thought out.
- Centralizing Lambda function names in config (modulo the default value) cleans up five controllers.
