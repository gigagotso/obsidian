# Deprecation audit — legacy channel columns on `conversation_reports`

> Goal: phase out the four legacy channel-attribution columns in favour of the single
> `conversation_reports.channel_line_id` FK (→ `channel_lines` → `channels`).
>
> Deprecated columns: **`from_website`**, **`from_hub`**, **`hub_provider`**, **`widget_id`**.

---

## TL;DR

| Legacy column | Type | Meaning | Replacement |
|---|---|---|---|
| `from_website` | bool | conversation originated on a website widget | `channel_line_id` → `channels.name = 'website'` |
| `from_hub` | bool | conversation came from a social hub channel | `channel_line_id` → `channels.name != 'website'` (or `from_website = 0`) |
| `hub_provider` | string | `messenger`/`whatsapp`/`telegram`/`viber`/`email`/`instagram`/`pbx` | `channel_line_id` → `channels.name` |
| `widget_id` | uuid (FK→`widgets`) | which website widget (set **only** for website convos) | `channel_line_id` (website line: `identifier = widget->id`, `name = widget->name`) |

**Key facts**
- `GenerateConversationReport` **already writes `channel_line_id`** alongside the legacy columns
  (lines 123/150/161/248/269/331), so the new column is populated going forward.
- `channel_line_id` was added in migration `2025_07_07_131636_add_channel_line_id_field_to_conversation_reports_table`.
  Pre-rollout rows have now been **backfilled** (done), so `channel_line_id` is populated across all
  history and reads/filters/labels can flip to it without losing attribution. ✅ Blocker resolved.
- The human-readable channel label (`"website"` / provider name) and channel-line label currently
  come from CASE expressions over these columns. Reproducing them from `channel_line_id` requires a
  join `channel_lines` → `channels` (`channels.name` = channel; `channel_lines.name` = line).

⚠️ **`widget_id` is heavily overloaded.** Only `conversation_reports.widget_id` is in scope. The
many other `widget_id` references (the `widgets` table PK, `conversations.extra_attributes->widget_id`,
`kbs.widget_id`, widget pivot tables, trigger conditions, observability middleware, cache keys) are
**unrelated and must not be touched** — see the exclusion list at the bottom.

---

## 1. `from_website`

| Location | Role | Notes / replacement |
|---|---|---|
| `database/migrations/2020_10_12_104418_create_conversation_reports_table.php:24` | schema | `boolean('from_website')` |
| `app/Models/ConversationReport.php:38` | model `$fillable` | keep until generator stops writing it |
| `app/Models/ConversationReport.php:109` | `$allowedFilterFields` | drop once filters migrate |
| `app/Actions/Reports/GenerateConversationReport.php:55,113,301` | **write/generate** | `DB::raw('if (hub_provider is null, 1, 0) as from_website')` — derived during report build |
| `app/Models/ConversationReport.php:427` | read (`getBasicFields` select) | replace with `channel_line_id` + join |
| `app/Models/ConversationReport.php:445,449` | **label** (CASE → `channel` & `channel_line`) | `channel`: `... = 'website'`; `channel_line`: `w.name` → `channel_lines.name` |
| `app/Models/ConversationReport.php:507,517` | read + label (`getInboxConversations`) | same as above |
| `app/Models/Traits/Filters/ConversationReports/Criteria/ChannelFilterTrait.php:17` | **filter** | `website` → `cr.from_website = 1`; replace with channel-line/channel join |
| `app/Actions/Reports/Traits/FiltersConversations.php:49` | **filter** (legacy `types[]`) | already being retired for analytics; superseded by `channel_ids[]`/`channel_line_ids[]` |
| `database/seeds/BulkDataSeeder.php:452` | seed data | test only |

## 2. `from_hub`

| Location | Role | Notes / replacement |
|---|---|---|
| `database/migrations/2020_10_12_104418_create_conversation_reports_table.php:25` | schema | `boolean('from_hub')` |
| `app/Models/ConversationReport.php:39` | model `$fillable` | |
| `app/Models/ConversationReport.php:110` | `$allowedFilterFields` | |
| `app/Models/ConversationReport.php:428` | read (`getBasicFields` select) | |
| `app/Actions/Reports/GenerateConversationReport.php:56,114,302` | **write/generate** | `DB::raw('if (hub_provider is not null, 1, 0) as from_hub')` |
| `app/Actions/Reports/Traits/FiltersConversations.php:52` | **filter** (legacy `types[] = hub`) | superseded by channel filters |
| `app/Console/Commands/ReopenMissedConversations.php:85` | **operational** (`where('from_hub', 1)`) | console command — migrate carefully or leave; not user-facing |
| `database/seeds/BulkDataSeeder.php:453` | seed data | test only |

## 3. `hub_provider`

| Location | Role | Notes / replacement |
|---|---|---|
| `database/migrations/2020_10_12_104418_create_conversation_reports_table.php:32` | schema | `string('hub_provider')->nullable()` |
| `app/Models/ConversationReport.php:40` | model `$fillable` | |
| `app/Models/ConversationReport.php:80` | filter `$fieldMapping['channel']` → `cr.hub_provider` | repoint to channel-line/channel join |
| `app/Models/ConversationReport.php:111` | `$allowedFilterFields` | |
| `app/Models/ConversationReport.php:429` | read (`getBasicFields` select) | |
| `app/Models/ConversationReport.php:446,518` | **label** (CASE `channel` ELSE branch) | replace with `channels.name` |
| `app/Models/ConversationReport.php:508` | read (`getInboxConversations`) | |
| `app/Actions/Reports/GenerateConversationReport.php:54,65,122,149,160,247,268,301,302,330` | **write/generate** | sourced from `conversations.extra_attributes->social_hub->provider->name`; also drives `from_website`/`from_hub` derivation; `:65` business logic (`hub_provider === 'messenger' && is_answered`) |
| `app/Models/Traits/Filters/ConversationReports/Criteria/ChannelFilterTrait.php:19` | **filter** | non-website channel → `cr.hub_provider = $value` |
| `app/Actions/Reports/Traits/FiltersConversations.php:46` | **filter** (legacy `types[]` providers) | superseded |
| `app/Actions/Reports/GetDailyCallSummaryReport.php:45` | **filter** (`where('cr.hub_provider', 'pbx')`) | PBX call summary — repoint to pbx channel line |
| `app/Http/Resources/ConversationResource.php:60-62` | read (`social_provider`) | API response field; needs a replacement source before removal |
| `app/Actions/Conversation/GetConversationHistory.php:96,118` | read (select/label) | conversation history payload |
| `database/seeds/BulkDataSeeder.php:454` | seed data | test only |

## 4. `widget_id` (the `conversation_reports.widget_id` column ONLY)

| Location | Role | Notes / replacement |
|---|---|---|
| `database/migrations/2020_10_12_104418_create_conversation_reports_table.php:33,44` | schema + FK→`widgets` | |
| `app/Models/ConversationReport.php:42` | model `$fillable` | |
| `app/Models/ConversationReport.php:78` | filter `$fieldMapping['channel_lines']` → `cr.widget_id` | **repoint to `cr.channel_line_id`** |
| `app/Models/ConversationReport.php:408` | `buildBaseQuery` `leftJoin('widgets as w', 'w.id', 'cr.widget_id')` | replace with join to `channel_lines` |
| `app/Models/ConversationReport.php:430,449` | read + channel_line label (`w.name`) | `channel_lines.name` |
| `app/Models/Traits/Filters/ConversationReports/Criteria/ChannelLineFilterTrait.php:13` | **filter** (`whereIn('cr.widget_id', $value)`) | this is the "channel line" filter today — **migrate to `cr.channel_line_id`** (note: current impl only matches website widgets) |
| `app/Actions/Reports/GenerateConversationReport.php:124,148,159,246,267,332,342` | **write/generate** | from `conversations.extra_attributes->widget_id`; `:342` joins `widgets` |
| `app/Actions/Conversation/GetConversationHistory.php:74` | read (select) | |

---

## 5. What has already migrated (done)

- **Agent/user analytics reports** (`/v1/analytics/users/{performance,response-times,satisfaction}`)
  and **general analytics** (`/v1/analytics`) now filter on `channel_line_id` via `channel_ids[]` /
  `channel_line_ids[]` (`FiltersConversations::filterChannels`). They no longer accept `widgets[]`,
  and the channel-level `types[]` enum values were removed. See `user-analytics-reports-plan.md`.
- **Conversation-report filters (step 1)** now target `channel_line_id`:
  - `ChannelLineFilterTrait::applyChannelLineFilter` → `whereIn('cr.channel_line_id', $value)`
    (was `cr.widget_id`).
  - `ChannelFilterTrait::applyChannelFilter` → `cr.channel_line_id IN (account's lines for that
    `channels.name`)` (was `from_website` / `hub_provider`).
  - `ConversationReport::$fieldMapping` `channel_lines`/`channel` repointed to `cr.channel_line_id`.
  - `FilterValueRule`: `channel_lines` added to integer-array validation (values are now channel-line
    IDs, not widget UUIDs).
  - ⚠️ **Contract change**: the `channel_lines` filter now expects **channel-line IDs**, not widget
    UUIDs. Clients must send channel-line IDs.
- **Conversation-report labels (step 2)** now prefer `channel_lines`/`channels`, with a
  `COALESCE`/CASE fallback to the legacy columns so output is identical even when `channel_line_id`
  is NULL (pre-backfill rows / local DB):
  - `getBasicFields`: `channel = COALESCE(ch.name, <legacy CASE>)`,
    `channel_line = CASE WHEN cr.from_website=1 THEN COALESCE(cl.name, w.name) ELSE NULL END`;
    added `channel_lines`/`channels` joins.
  - `getInboxConversations`: same `COALESCE(ch.name, …)` for `channel` + the joins.
  - The legacy columns are still **selected** (raw `from_website`/`hub_provider`/`widget_id`) and the
    join fallback is retained — so dropping them is still a Tier-3 step.
- **Shared resolver (refactor C)**: `ChannelLine::idsForChannelIds()` / `idsForChannelNames()` now
  back both the analytics filter (`FiltersConversations::filterChannels`) and the conversation-report
  `ChannelFilterTrait`, removing the duplicated channel_lines sub-query logic.
- **Dead-code cleanup (B)**: removed the duplicate `avg_response_time` in `GetUserResponseTimesReport`,
  the unused `user` param in `AnalyticsController::index`, and the never-consumed
  `ConversationReport::$fieldMapping`.

- **Backward-compat removed from the report query layer (Tier-3 read side)**: the legacy
  fallbacks/columns are gone from all report queries.
  - `ConversationReport::getBasicFields` — dropped raw `from_website`/`from_hub`/`hub_provider`/`widget_id`
    selects and the `widgets` join; `channel = ch.name`, `channel_line = cl.name` (no COALESCE/CASE
    fallback; `channel_line` now populated for **all** channels, not website-only). Added `cr.channel_line_id`
    to the select.
  - `ConversationReport::getInboxConversations` — dropped the raw `from_website`/`hub_provider` selects;
    `channel = ch.name`.
  - `ConversationReport::$allowedFilterFields` — removed the dead `from_website`/`from_hub`/`hub_provider`
    entries (they had no dispatch case — silent no-ops).
  - `FiltersConversations::filterConversationTypes` — removed the channel branches
    (`hub_provider`/`from_website`/`from_hub`); now message-content only (`audio/text/comment/file`).
  - `GetDailyCallSummaryReport` — `cr.hub_provider = 'pbx'` → `cr.channel_line_id IN (pbx lines)` via
    `ChannelLine::idsForChannelNames(['pbx'], …)`.
  - ⚠️ **Breaking**: the detailed/inbox report responses no longer include the raw
    `from_website`/`hub_provider`/`widget_id` fields; `channel`/`channel_line` now derive purely from
    `channel_line_id`, so rows without it return NULL (fine on backfilled production; this local DB is
    ~99.85% NULL).

### Still TODO
- **Step 3 non-report readers** (still read legacy columns, left intact — emit a frontend-consumed
  field with nuanced NULL semantics): `ConversationResource.social_provider`, `GetConversationHistory`
  (`report.hub_provider`/`report.widget_id`), `ReopenMissedConversations` (`from_hub`).
- **Schema cutover (final)**: stop writing the legacy columns in `GenerateConversationReport`, remove
  them from `$fillable`, then drop the columns via migration. Requires confirming nothing else reads
  them and that the production backfill is complete.

## 6. Recommended migration order (reads/filters → writes)

1. **Filters first** (low risk, high value): repoint `ConversationReport::$fieldMapping`
   (`channel_lines` → `cr.channel_line_id`, `channel` → resolved via `channels.name`),
   `ChannelLineFilterTrait` → `cr.channel_line_id`, `ChannelFilterTrait` → channel-line/channel join.
2. **Labels/reads**: replace the CASE expressions and the `widgets` join in
   `getBasicFields`/`getInboxConversations` with a `channel_lines`/`channels` join
   (`channels.name` as channel, `channel_lines.name` as channel_line).
3. **Other readers**: `ConversationResource.social_provider`, `GetConversationHistory`,
   `GetDailyCallSummaryReport` (pbx), `ReopenMissedConversations`.
4. **Stop writing** the legacy columns in `GenerateConversationReport` once (1)–(3) ship. ✅ The
   backfill prerequisite is already complete.
5. **Drop columns** via migration last.

> **Backfill: done.** `channel_line_id` is populated across all history, so there is no remaining
> NULL-attribution blocker for steps 4–5.

---

## Appendix — `widget_id` references that are OUT OF SCOPE (do not touch)

These are not the `conversation_reports.widget_id` column:

- `widgets` table PK / `extra_attributes->widget_id` on `conversations`:
  `Conversation.php:161`, `Widget.php:139`, `Visitor.php:342`, `Widgets/ConversationsController.php`,
  `Widgets/RTCRobotController.php`
- KB: `Kb.php:25`, `KbResource.php`, `Widgets/KbResource.php`, `KbService.php`,
  `UpdateKbGeneralRequest.php`, migration `2026_03_08_000001_create_kbs_table.php`
- Widget pivots: `WidgetFieldPivot.php`, `WidgetFeaturePivot.php`, their observers, and migrations
  `2020_01_14_..._widget_has_fields`, `2019_10_06_..._widget_has_features`, `2019_11_25_..._widget_has_queues`
- Triggers / chatbots: `Triggers/Conditions/WidgetName.php`, `TriggersController.php`,
  `TriggerResource.php`, `ChatBots/Conditions/RoutingCondition.php`, `Rules/CheckTrigger*.php`,
  `ChatBotService.php`, `UnresponsiveConversationService.php`, `DepartmentResource.php`
- Infra: `ObservabilityMiddleware.php`, `Actions/Widget/ClearCache.php`,
  `Services/Widget/WidgetManagementService.php`
