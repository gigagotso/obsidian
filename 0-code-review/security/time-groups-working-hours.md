# Time Groups (Working Hours) – API + Logic + Security Notes

This document describes how **working hours** are modeled and evaluated via `App\Models\TimeGroup`, which APIs mutate/expose them, where they are used across the system (departments/queues, reports, triggers, chatbots), and what to improve from a security/validation standpoint (including an **OWASP Top 10** mapping).

## Data Model

Time groups are stored in the `time_groups` table:

- **`id`**: integer primary key
- **`account_id`**: owning account
- **`name`**: display name (unique per account via controller validation)
- **`opening_hours`**: JSON (required column)
- **`timezone`**: string (defaults to `config('app.timezone')`)
- **`extra_attributes`**: JSON nullable (schemaless)

Relevant migrations:

- `database/migrations/2019_04_21_155731_create_time_groups_table.php`
- `database/migrations/2019_07_04_135511_add_timezone_to_time_groups_table.php`
- `database/migrations/2020_07_03_180821_add_extra_attributes_to_time_groups_table.php`

## Storage Format vs API Format

### Storage format (`TimeGroup.opening_hours`)

Internally `opening_hours` is stored as a JSON object where each weekday maps to an **array of time-range strings**:

```json
{
  "monday":   ["09:00-17:00"],
  "tuesday":  ["09:00-17:00"],
  "wednesday":["09:00-17:00"],
  "thursday": ["09:00-17:00"],
  "friday":   ["09:00-17:00"],
  "saturday": [],
  "sunday":   [],
  "exceptions": {
    "2025-12-25": []
  }
}
```

Notes:

- Each day can have **multiple ranges** (e.g. `["09:00-12:00","13:00-17:00"]`).
- `exceptions` are stored as an object whose keys are dates in `Y-m-d` and values are arrays (currently always `[]`).

This storage shape is produced by `TimeGroup::setOpeningHoursAttribute(...)`, which converts API-style input into these `"from-to"` strings.

### API format (what you send/receive)

The API returns `opening_hours` in a “formatted” structure:

```json
{
  "monday": { "hours": [ { "from": "09:00", "to": "17:00" } ] },
  "tuesday": { "hours": [ { "from": "09:00", "to": "17:00" } ] },
  "exceptions": ["2025-12-25"]
}
```

This is produced by:

- `TimeGroup::getOpeningHoursFormattedAttribute()` (weekday ranges are split into `from`/`to`)
- `TimeGroupResource` (returns `opening_hours`, `opening_hours_merged`, `timezone`, etc.)

## Core Business Logic

### “Is the time group open?”

`TimeGroup::getWorkingHours()` creates a `Spatie\OpeningHours\OpeningHours` object using:

- the stored `opening_hours` JSON
- the `timezone` field

Then callers check:

- `->isOpen()` / `->isClosed()`
- `->nextOpen(now())` / `->nextClose(now())`

### Merged display (“Opening Hours Merged”)

`TimeGroup::opening_hours_merged` groups consecutive weekdays that have identical hours into a compact representation used for UI display.

### Default time group on new accounts

`App\Listeners\Account\PrepareNewAccount` creates a `default` time group for new accounts.

## Related / Downstream Usage (Where working hours matter)

### Departments (Queues)

Departments are implemented as `App\Models\Asterisk\Queue` (“department” resource).

Key logic:

- `Queue::isOpen()`:
  - true if `time_group_id` is set and either:
    - `timeGroup->getWorkingHours()->isOpen()` OR
    - `timeGroup->extra_attributes->get('always_open', false)` is truthy

API exposure:

- `DepartmentResource` includes (when `timeGroup` relationship is loaded):
  - `time_group` (full `TimeGroupResource`)
  - `is_open`
  - `next_open` / `next_close` (wrapped with `rescue(...)` to avoid crashing on invalid configs)

### Reports (Conversation Reports)

`GenerateConversationReport` sets `during_working_hours` based on whether the department is open at report-generation time:

- `Queue::find($department_id)->isOpen()`

Filtering:

- `DuringWorkingHoursFilterTrait` filters reports by `cr.during_working_hours`.

### Triggers

- `Triggers/Conditions/DepartmentIsOpen` returns `optional($conversation->department)->isOpen()`.

### Chatbots

- `ChatBots/Conditions/BusinessHoursCondition` loads a `TimeGroup` by raw id from payload and checks `isOpen()` / `isClosed()`.

## Allowed APIs (Routes) + Required Access

All of the below are under `Route::prefix('v1')->middleware(['auth:api'])` in `routes/api.php`.

### Time Group endpoints

Resource endpoints (`Route::apiResource('time-groups', TimeGroupsController::class)`):

- **GET** `v1/time-groups` (list)
- **POST** `v1/time-groups` (create)
- **GET** `v1/time-groups/{time_group}` (show)
- **PUT/PATCH** `v1/time-groups/{time_group}` (update name/timezone)
- **DELETE** `v1/time-groups/{time_group}` (delete)

Custom endpoints:

- **POST** `v1/time-groups/{time_group}/clone` (clone an existing time group)
- **PATCH** `v1/time-groups/{time_group}/days` (update weekday hours)
- **PATCH** `v1/time-groups/{time_group}/exceptions` (update exception dates)

### Scopes/permissions (important nuance)

`TimeGroupsController` applies Passport scope middleware only to:

- `store` → `scope:time_groups:create`
- `index`, `show` → `scope:time_groups:read`
- `update` → `scope:time_groups:update`
- `destroy` → `scope:time_groups:delete`

`TimeGroupPolicy` enforces:

- **permissions** (`time_groups:*`) AND
- **account ownership** (time group must belong to user’s account)

However:

- The custom controller methods **`clone`**, **`updateDays`**, **`updateExceptions`** are **not** included in the scope middleware list.
- `authorizeResource(TimeGroup::class)` typically covers only the conventional resource methods (`index`, `show`, `store`, `update`, `destroy`) unless custom authorization is explicitly added.

That means the custom endpoints may currently be missing one or both of:

- explicit **scope checks**
- explicit **policy authorization** for the given `$time_group`

This should be reviewed and tightened (see recommendations below).

### Department endpoints related to time groups

Departments require a `time_group_id` on create/update (`DepartmentsController@store` / `@update`).

Important nuance:

- The validation uses `exists(TimeGroup::class, 'id')` but does **not** restrict the `time_group_id` to the current account. This can allow associating a department to another account’s time group if the id is guessable.

## Input Validation / Sanitization (What exists today)

### Time group create/update (name/timezone)

Validated in `TimeGroupsController`:

- `name`: required string, max 255, unique per account
- `timezone`: required string, `timezone` rule, max 255

### Update weekday hours (`PATCH .../days`)

Validated structure:

- `{weekday}`: `nullable|array|min:1`
- `{weekday}.hours`: `nullable|array|min:1`
- `{weekday}.hours.*.from` / `.to`: required_with counterpart, `multi_date_format:H:i,H:i:s`

Notes:

- `multi_date_format` is defined in `AppServiceProvider` and accepts either `H:i` or `H:i:s`.
- There is **no validation** today that:
  - `from < to`
  - ranges don’t overlap
  - ranges are within `00:00`–`23:59`
  - a day is omitted vs “explicitly closed”

### Update exceptions (`PATCH .../exceptions`)

Validated structure:

- `exceptions`: `nullable|array`
- `exceptions.*`: `date_format:Y-m-d`

Notes:

- No uniqueness constraint (duplicate dates are possible in the request).
- No max size constraint (large arrays could be used for resource pressure).

### Model-level “sanitization” / normalization

`TimeGroup::setOpeningHoursAttribute(array $value)`:

- Converts weekday input into the canonical `"from-to"` strings.
- For `exceptions`, converts a list into `{ "YYYY-MM-DD": [] }`.
- If a day payload does not match the expected `['hours' => ...]` shape, it defaults that day to `00:00-23:59` (always open), which can mask malformed input.

## Security Review (OWASP Top 10 – Relevant Items)

This section maps time group/working-hours behavior to OWASP Top 10 (2021) categories.

### A01: Broken Access Control

High relevance:

- **Custom time-group endpoints** (`clone`, `days`, `exceptions`) may be missing explicit policy authorization and/or scope middleware.
- `DepartmentsController` allows `time_group_id` that exists globally, not scoped to the authenticated account.
- `BusinessHoursCondition` loads a `TimeGroup` by raw ID with no account scoping, which is an IDOR risk if payloads can be influenced by a tenant user.

Recommended mitigations:

- Enforce `authorize('update', $time_group)` (or appropriate ability) in `updateDays`, `updateExceptions`, `clone`.
- Add scope middleware for those methods consistent with `update` (and `create` for clone).
- Validate `time_group_id` with an `exists` clause constrained by `account_id`.
- Refactor chatbot condition to resolve time groups through the bot/account context (or store `time_group_id` on the bot and scope it).

### A04: Insecure Design

Medium relevance:

- The system calculates “during_working_hours” for reports using *current* open/closed status at report time. If the intended definition is “open at conversation time”, this design can produce incorrect analytics.

Mitigations:

- Decide on the intended semantic and compute using the conversation timestamp + department’s time group at that time.
- Consider storing a snapshot of effective hours (or time_group_id + timezone) on report rows if historical correctness matters.

### A05: Security Misconfiguration

Medium relevance:

- Missing middleware application (scopes/authorization) on non-resource controller actions is a common misconfiguration.

Mitigations:

- Standardize with a helper that applies scope + authorization to custom actions.

### A06: Vulnerable and Outdated Components

Low–Medium relevance:

- Working hours rely on `spatie/opening-hours`. Keep dependency pinned and monitored for security updates.

### A07: Identification and Authentication Failures

Low relevance (handled by `auth:api`), but ensure:

- token scopes are consistently required for all sensitive actions.

### A09: Security Logging and Monitoring Failures

Low–Medium relevance:

- Consider logging admin changes to time groups (days/exceptions) if used for call routing / operational behavior.

### A10: Server-Side Request Forgery (SSRF)

Not directly relevant to time groups as implemented.

## Concrete Optimization / Hardening Checklist

### Authorization & scopes

- Apply scope middleware to:
  - `clone` → `scope:time_groups:create` (or `time_groups:update` depending on policy)
  - `updateDays` → `scope:time_groups:update`
  - `updateExceptions` → `scope:time_groups:update`
- Add explicit policy checks in those methods (e.g. `$this->authorize('update', $time_group)` or `$this->authorize('create', TimeGroup::class)` for clone).

### Validation improvements

- Add a custom validator to ensure:
  - each range has `from < to`
  - ranges do not overlap per day
  - optional: normalize/round to minutes
- Add size limits:
  - max number of ranges/day
  - max number of exceptions

### Tenant scoping fixes

- In `DepartmentsController`, constrain `time_group_id` to the authenticated account:
  - e.g. `Rule::exists('time_groups', 'id')->where('account_id', $accountId)`
- In `BusinessHoursCondition`, ensure the time group belongs to the same account context as the bot/workflow.

### Data normalization / correctness

- Consider changing `setOpeningHoursAttribute` to:
  - reject unexpected day shapes instead of silently defaulting to `00:00-23:59`
  - consistently persist empty arrays for closed days

## Request/Response Examples

### Create a time group

Request:

```json
{
  "name": "Support hours",
  "timezone": "America/New_York"
}
```

Response includes:

- `opening_hours` (formatted)
- `opening_hours_merged` (display-friendly)

### Update weekday hours

Request (partial update; omitted days remain unchanged):

```json
{
  "monday": {
    "hours": [
      { "from": "09:00", "to": "12:00" },
      { "from": "13:00", "to": "17:00" }
    ]
  },
  "tuesday": {
    "hours": [
      { "from": "09:00", "to": "17:00" }
    ]
  }
}
```

### Update exceptions

Request:

```json
{
  "exceptions": ["2025-12-25", "2025-01-01"]
}
```


