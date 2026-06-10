# Notification Preferences — Architecture Redesign

**Status:** Proposal / design doc
**Scope:** Backend (Laravel API) + Frontend (settings page)
**Author:** Engineering
**Date:** 2026-06-05

> **Terminology note.** This doc deliberately avoids the word *channel* for notification delivery.
> In LiveCaller, **channel** already means a conversation/integration source (web widget, Facebook,
> WhatsApp, …). To prevent collisions, the delivery dimension (email, browser, desktop, sound,
> mobile) is called a **delivery method** (key: `delivery_method` / `methods`). The only place
> "channel" still appears is when referring to Laravel's *framework* notification drivers
> (`mail`, `broadcast`, `database`) — the framework's own term — and those mentions are explicitly
> flagged.

---

## 0. TL;DR

We store notification preferences as a few ad-hoc keys inside a generic `users.settings` JSON
column, mixed with unrelated settings (`sip.auto_register`, `is_away`, `notification_sounds`).
The dispatch layer (`NotificationService`) only honors preferences for **one** event
(`conversation_started`), and its per-event delivery-method list is hardcoded in PHP. This does not
scale to the event types and delivery methods we actually want (assignment, transfer, mention, SLA,
missed, desktop/mobile push, quiet hours, intervals).

**Recommendation:** drive everything from a **server-defined catalog** of events and delivery
methods, resolve preferences through a single storage-agnostic resolver, and store user overrides as
**structured JSON now** (graduating to a *single* **`notification_preferences`** table — one row per
`(user, event)` with the delivery methods stored as a JSON object in that row, **not** a row per
method — only when a querying feature actually justifies it; see §4.6). Keep non-notification
settings (`sip`, `is_away`) where they are. This gives us per-method control, quiet hours, intervals,
defaults, and the ability to add events/methods without a migration or a frontend rewrite.

---

## 1. Problems With the Current Structure

### 1.1 What exists today

Preferences live in `users.settings` (a `json` column — `database/migrations/2014_10_12_000000_create_users_table.php:23`)
and are read/written through the `HasSettings` trait (`app/Models/Traits/HasSettings.php`).

```json
{
  "sip": { "auto_register": true },
  "is_away": false,
  "notifications": {
    "conversation_started": { "email": true },
    "active_idle_conversation": { "browser": true, "interval": 1 }
  },
  "notification_sounds": true
}
```

Write path: `UserController::updateNotifications` (`app/Http/Controllers/UserController.php:281`)
validates a fixed shape and overwrites the whole `notifications` key.

Read path: `UserResource` (`app/Http/Resources/UserResource.php:115`) returns
`settings.notifications` verbatim, plus `notification_sounds` and `sip.auto_register`.

Dispatch path: `NotificationService::process()` (`app/Services/NotificationService.php:25`) has a
hardcoded `eventsAliasMap` listing exactly one event and checks
`settings.notifications.{event}.{delivery_method}` before notifying.

### 1.2 Concrete problems

1. **Unrelated concerns share one blob.** `sip`, `is_away`, `notification_sounds`, and
   `notifications` all live in the same `settings` object. They have nothing to do with each other,
   change at different times, and are validated/owned by different controller methods. The JSON
   object is a junk drawer.

2. **`notification_sounds` is a global boolean, detached from the matrix.** Sound is conceptually a
   *delivery method* (like browser/email), but it's modeled as a single on/off flag outside the
   per-event structure. You cannot say "play a sound for mentions but not for new conversations."

3. **The dispatch layer ignores almost all preferences.** `eventsAliasMap` contains only
   `conversation_started`. `active_idle_conversation` preferences are *stored by the UI* but **never
   read by any dispatcher** — the toggle in the screenshot does nothing on the backend today.
   Other notifications (`ConversationTransferred`, `CallbackRequested`, …) are sent directly,
   bypassing preferences entirely.

4. **Delivery-method mapping is hardcoded and brittle.** `process()` hand-maps `email → mail`,
   switches on `mail/database/broadcast` (Laravel's framework channel drivers), and string-matches
   method names. Adding a delivery method means editing a `switch`. There is no single registry of
   "what delivery methods exist."

5. **No defaults model.** A user who has never opened the settings page has `settings = null` →
   every `setting('notifications.x.y', false)` returns `false` → they get **nothing**. New events we
   add are silently off for all existing users. There is no notion of a system default that a user
   overrides.

6. **No partial updates, race-prone writes.** `updateNotifications` rebuilds and overwrites the
   entire `notifications` subtree from a fixed payload. Two tabs saving different sections clobber
   each other. Adding a field means the validation rule, the rebuild block, and the resource must
   all be touched in lockstep.

7. **Not queryable.** "Which users want email on SLA warnings?" requires scanning a JSON column in
   PHP. We can't index it, can't join, can't aggregate, can't build a digest worker efficiently.

8. **No room for quiet hours, scheduling, digests, or per-method intervals.** The shape only
   supports `bool` (+ one `interval` hack bolted onto `active_idle_conversation`).

9. **No extensibility contract.** There's no schema/catalog of valid events × delivery methods, so
   the frontend hardcodes the two rows it knows about, and adding an event requires coordinated
   frontend + backend code changes.

---

## 2. Recommended Notification Architecture

The mental model used by Slack, Discord, Teams, Intercom, Linear, GitHub, etc. is a **matrix**:

```
                 Delivery methods →
Events ↓     Desktop  Mobile  Email  Browser  Sound
──────────────────────────────────────────────────
new convo      ☐       ☐       ☑       ☐       ☑
assigned       ☑       ☑       ☐       ☑       ☑
mention        ☑       ☑       ☑       ☑       ☑
SLA warning    ☑       ☑       ☑       ☐       ☑
...
```

(SIP is intentionally absent — it's a softphone connectivity setting, not a delivery method. See §8.)

Layered model:

1. **Catalog (code-owned, not in DB):** the canonical list of **events** and **delivery methods**,
   plus per-event metadata — which methods are *applicable*, the *default* state per method, whether
   the event supports an *interval*, its category/group, label, and description. This is the
   contract both the dispatcher and the frontend read.

2. **System defaults:** derived from the catalog. Used when a user has no explicit override.

3. **User overrides (source of truth):** sparse entries — "user 42 set `mention.desktop = on`."
   Absence of an entry = "use the default." (Stored as structured JSON now, graduating to rows; see §3.)

4. **Global modifiers:** quiet hours, "pause all", DND while available/away. Applied at send time
   *after* the matrix lookup.

5. **Resolution at send time:**
   `effective(user, event, method) = userOverride ?? catalogDefault`, then gated by
   global modifiers (quiet hours, DND, `is_away`), then by method applicability.

---

## 3. JSON vs. Dedicated Tables — Deep Dive

This is the central decision, so it gets the most space. The short version: **both work; the right
choice depends on whether anything other than "the user editing their own page" ever needs to read
these preferences.** Below are concrete shapes for each, how reads/writes look, and an honest
pros/cons grounded in LiveCaller's actual scale (a handful of events, modest agents-per-account, no
digest workers today).

### 3.1 What the JSON would actually look like

The current blob mixes everything. A *cleaned-up* JSON approach keeps notifications in their own
self-describing document. Two viable shapes:

**Option A — flat `event.method` map (closest to today, minimal change):**

```json
{
  "notifications": {
    "version": 2,
    "events": {
      "conversation_started":     { "email": true,  "desktop": false, "browser": false, "sound": true },
      "conversation_assigned":    { "email": false, "desktop": true,  "browser": true,  "sound": true },
      "active_idle_conversation": { "browser": true, "sound": true, "interval": 3 },
      "mention":                  { "email": true,  "desktop": true,  "browser": true,  "sound": true },
      "sla_warning":              { "email": true,  "desktop": true,  "sound": true }
    },
    "global": {
      "sounds_enabled": true,
      "dnd_enabled": false,
      "quiet_hours": { "enabled": false, "start": "22:00", "end": "08:00", "timezone": "Asia/Tbilisi" },
      "mute_until": null
    }
  }
}
```

**Option B — explicit methods block + per-event overrides (more structured, self-documenting):**

```json
{
  "notifications": {
    "version": 2,
    "global": {
      "sounds_enabled": true,
      "dnd_enabled": false,
      "quiet_hours": { "enabled": false, "start": "22:00", "end": "08:00", "timezone": "Asia/Tbilisi" }
    },
    "events": {
      "active_idle_conversation": {
        "methods": { "browser": true, "sound": true },
        "interval": 3
      },
      "mention": {
        "methods": { "email": true, "desktop": true, "browser": true, "sound": true }
      }
    }
  }
}
```

Key design notes for the JSON approach:

- **Store only overrides, not the full matrix.** A user who never touched a cell has *no key* for
  it; the resolver falls back to the catalog default (§5). This is how you get real "unset vs off"
  semantics in JSON — the same thing a missing table row gives you. (Storing `false` explicitly for
  every cell loses that distinction and means new events default to off — the current bug.)
- **`version` field** lets you migrate the shape later without guessing.
- **Validate against the catalog on write** (events/methods/intervals must exist), so the blob can't
  drift into garbage.
- Resolution stays identical to the table design:
  `effective = userJson[event][method] ?? catalogDefault[event][method]`.

### 3.2 What the same data looks like in tables

**Recommended table shape (Combo 2, §4.6) — one row per `(user, event)`, methods in a JSON payload:**

```
notification_preferences
  user_id | event                     | preferences (JSON)
  --------+---------------------------+------------------------------------------------
       42 | conversation_started      | { "methods": { "email": true } }
       42 | active_idle_conversation  | { "methods": { "browser": true }, "interval": 3 }
       42 | mention                   | { "methods": { "email": false } }   ← explicit "off"

-- globals (sounds/DND/quiet hours) stay in users.settings.notifications.global
```

A user who customized 3 events → 3 rows total. Absent row = all-default.

<details>
<summary>For contrast — the maximally-normalized shape (§4.1, one row per method — overkill):</summary>

```
notification_preferences          notification_event_settings    notification_user_settings
 user | event       | method | on   user | event      | interval   user | sounds | dnd | quiet...
 -----+-------------+--------+---   -----+------------+--------     -----+--------+-----+-----
   42 | conv_started| email  | 1      42 | idle_conv  | 3
   42 | mention     | email  | 0
   42 | mention     | desktop| 1     ← one row PER method → 3-table split, 40+ rows/user
```
This is why §4.1 is *not* the default — same information, many more rows and tables.
</details>

### 3.3 Reads and writes, side by side

| Operation | JSON | Tables |
|---|---|---|
| **Load one user's page** | `SELECT settings` → 1 column, parse in PHP | `SELECT * FROM notification_preferences WHERE user_id = ?` (+2 small selects) |
| **Save one toggle** | read blob → `data_set('notifications.events.mention.email', true)` → write whole column | `INSERT ... ON DUPLICATE KEY UPDATE enabled = ?` (one row) |
| **Dispatch to one known user** | parse their blob, look up `event.method` | one indexed lookup, or load all their rows once |
| **"Who wants email on SLA?" (fan-out / digest)** | load **every** user row, JSON-parse, filter in PHP | `SELECT user_id FROM notification_preferences WHERE event='sla_warning' AND delivery_method='email' AND enabled=1` |
| **Concurrent saves from two tabs** | last write wins, clobbers the other tab's section | independent rows, no clobber |
| **Add a new event** | nothing — catalog handles defaults, blob stays sparse | nothing — catalog handles defaults, rows stay sparse |

Note the bottom two rows are the *only* places the two approaches meaningfully diverge in practice.
Everything else is a wash at your scale.

### 3.4 JSON — pros and cons

**Pros**
- **Zero migrations.** Adding events/methods/quiet-hours is a catalog + validation change; the
  column never changes shape.
- **One read, one write** for the by-far-most-common operation (a user editing their own page).
  Already wired through the `HasSettings` trait and `UserResource`.
- **Trivially matches the API payload.** The frontend basically gets the document it edits.
- **Lowest implementation cost** — reuses everything that exists today.
- **Plenty fast** for per-user reads even at large scale (it's one indexed row by PK).

**Cons**
- **Cross-user queries are O(all users).** Any feature that asks "which users want delivery method X
  for event Y" must load and parse every user's blob. Digest emails, batched push fan-out, analytics
  ("how many agents enabled SLA alerts?") all hit this wall.
- **No indexing / joins / aggregation** on the contents.
- **Concurrency is last-write-wins** unless you do read-modify-write carefully (the current code
  overwrites the whole `notifications` subtree — clobber-prone).
- **Schema discipline is on you.** Without strict catalog validation on every write, the blob can
  silently drift (typo'd keys, stale events). Tables get this from a `UNIQUE` constraint + FK for
  free.
- **No natural history/audit.** You'd diff blobs; tables give you `updated_at` per cell and an easy
  audit hook.
- **MySQL JSON ergonomics are mediocre** (PHP 7.4 / older Laravel): generated columns or
  `JSON_EXTRACT` indexes are possible but clunky compared to plain columns.

### 3.5 Tables — pros and cons

**Pros**
- **Queryable.** Digest/fan-out workers become one indexed `WHERE` instead of a full scan. This is
  the entire reason large apps normalize.
- **Real per-cell concurrency** via upsert — no clobber.
- **Constraints enforce correctness** (`UNIQUE(user, event, delivery_method)`, FK cascade on user
  delete).
- **Per-cell history/audit** is natural (`updated_at`, optional audit table).
- **Cleanly separates** notification state from the `settings` junk drawer.

**Cons**
- **More moving parts.** Three tables, models, relations, a resolver, and a backfill migration.
- **Page load is a few small queries** instead of one column (negligible, but non-zero).
- **The API still needs to assemble a nested document** from rows — you don't escape building the
  JSON shape for the frontend; you just build it from rows instead of reading it directly.
- **Migration cost now** for benefits you may not use until later.
- **Over-normalization risk:** three tables for a handful of events can feel heavy if quiet
  hours/digests never ship.

### 3.6 Performance reality check (your scale)

- **Per-user page read/write:** indistinguishable between the two. Both are a single-digit-ms
  indexed operation. Don't choose based on this.
- **The only operation that actually separates them is cross-user fan-out.** Today your dispatch is
  per-event to a known set of `account->users` (see `ConversationCreated`), which you'd iterate
  anyway — so even fan-out doesn't currently require querying *by preference*. You'd need that query
  only when you add **digests** or **push batching across many users**.
- Conclusion: at present volumes, **JSON has no performance problem.** The case for tables is
  *future queryability and concurrency correctness*, not speed.

### 3.7 Decision matrix

| If you... | Lean |
|---|---|
| Want the smallest correct fix and have no digest/cross-user-query plans | **JSON (structured + catalog)** |
| Edit prefs almost exclusively from the user's own settings page | **JSON** |
| Plan email digests, batched push, or "notify all agents who opted into X" workers | **Tables** |
| Need audit history of every preference change | **Tables** |
| Are unsure but want to keep the door open | **Hybrid (§3.8)** |

### 3.8 Hybrid (keep the door open)

If you want JSON's simplicity now without burning the bridge to tables:

1. Build the **catalog + resolver + delivery-method registry** first (§5, §12 Phase 1). This is the
   part that actually fixes the bugs, and it's **storage-agnostic** — the resolver reads
   `userPref ?? default` regardless of whether `userPref` comes from JSON or a row.
2. Store overrides in the cleaned-up **JSON** (§3.1) for now.
3. The day you need cross-user queries, add `notification_preferences` and **dual-write** (JSON +
   table) for one release, then flip the resolver's read source to the table. Because the resolver
   is the only reader, this is a localized change — not a rewrite.

This is the recommended path given LiveCaller's scale: **JSON storage, catalog-driven resolution,
table-ready architecture.** You get the clean design and the bug fixes immediately, and pay for
tables only when a feature actually needs them.

### 3.9 Recommendation

- **Source of truth:** start with **structured JSON** (Option A or B), validated against the catalog,
  storing only overrides. It fixes every concrete problem in §1 with zero migrations.
- **Architect storage-agnostically** so the resolver/registry don't care where overrides live.
- **Graduate to tables** (§4) only when you ship digests, cross-user batching, or audit — at which
  point the migration is localized behind the resolver.
- Either way, **keep `sip` and `is_away` out of it** — they're not notifications.

---

## 4. Suggested Database Schema (migration target)

This is the shape you migrate *toward* when a querying feature justifies tables (§3.8). The catalog
(events, delivery methods) lives in **code** (a PHP enum/config), not in tables — it's deploy-time
constant and we don't want migrations to add an event. Only **user state** is persisted.

> **Read §4.6 first.** The recommended target is **Combo 2** — a *single* table with one row per
> `(user, event)` and the delivery methods stored as a JSON object in that row. The §4.1–4.3 tables
> below are the **maximally-normalized reference** — the heaviest option, kept for completeness and
> for the rare case you need cell-level SQL. **§4.1's one-row-per-delivery-method is overkill for our
> scale** (up to events × methods rows per user, e.g. ~40, to express a tiny config) and is **not**
> the default. If you skim one subsection, skim §4.6.

### 4.1 Per-delivery-method overrides (maximally normalized — usually overkill)

> ⚠️ **One row per `(user, event, delivery_method)`.** This is the heaviest layout: a user who
> customizes one event across 5 methods produces 5 rows. Its only real advantage over Combo 2 (§4.6)
> is plain-column SQL on a single method — which a digest worker rarely needs and Combo 2 can do via
> a JSON-path generated column anyway. **Prefer Combo 1 (no rows) or Combo 2 (one row per event).**
> Use this only if you have a concrete need to query/index individual method cells in SQL.

```sql
CREATE TABLE notification_preferences (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    event           VARCHAR(64)  NOT NULL,   -- 'conversation_started', 'mention', ...
    delivery_method VARCHAR(32)  NOT NULL,   -- 'email','browser','desktop','mobile','sound'
    enabled         TINYINT(1)   NOT NULL,   -- explicit override (on/off)
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,

    UNIQUE KEY uniq_user_event_method (user_id, event, delivery_method),
    KEY idx_event_method_enabled (event, delivery_method, enabled),   -- digest/worker lookups
    CONSTRAINT fk_np_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);
```

Row present → explicit user choice. Row absent → fall back to catalog default. This is the entire
"matrix." A user with all-defaults has **zero rows** (cheap).

### 4.2 Per-event settings (interval, etc.)

Intervals and similar per-event scalars don't belong on a per-method row. Keep them separate:

```sql
CREATE TABLE notification_event_settings (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    event           VARCHAR(64) NOT NULL,
    interval_value  SMALLINT UNSIGNED NULL,   -- e.g. minutes for active_idle_conversation
    sound_key       VARCHAR(32) NULL,         -- which sound for this event (see §8.1); NULL = default
    -- room to grow: threshold, snooze_until, etc.
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    UNIQUE KEY uniq_user_event (user_id, event),
    CONSTRAINT fk_nes_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);
```

### 4.3 Global per-user notification settings (quiet hours, master switches)

These are one-row-per-user and genuinely "read as a unit" — they can stay in JSON (`settings`) **or**
get a small table. A table keeps them queryable for the send-time gate:

```sql
CREATE TABLE notification_user_settings (
    user_id            BIGINT UNSIGNED PRIMARY KEY,
    sounds_enabled     TINYINT(1)  NOT NULL DEFAULT 1,   -- replaces settings.notification_sounds
    dnd_enabled        TINYINT(1)  NOT NULL DEFAULT 0,   -- master "pause all"
    quiet_hours_enabled TINYINT(1) NOT NULL DEFAULT 0,
    quiet_hours_start  TIME        NULL,                 -- e.g. 22:00
    quiet_hours_end    TIME        NULL,                 -- e.g. 08:00 (wraps midnight)
    timezone           VARCHAR(64) NULL,                 -- for quiet-hours eval
    mute_until         TIMESTAMP   NULL,                 -- temporary snooze
    created_at         TIMESTAMP NULL,
    updated_at         TIMESTAMP NULL,
    CONSTRAINT fk_nus_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);
```

### 4.4 Relationships

```
accounts ─1──*─ users ─1──*─ notification_preferences      (event × delivery-method cells)
                       ─1──*─ notification_event_settings   (per-event interval, etc.)
                       ─1──1─ notification_user_settings    (quiet hours / DND / sounds)
```

Eloquent: `User hasMany NotificationPreference`, `hasMany NotificationEventSetting`,
`hasOne NotificationUserSetting`.

> Three tables is the *maximally normalized* shape. It is almost certainly more than you need —
> §4.6 brainstorms how to collapse it.

### 4.6 Storage layout — how few tables can we actually get away with?

You're right that three tables is heavy. The split exists only because the maximally-normalized view
treats every cardinality separately. But the data is really just **three buckets**, and we can place
each independently:

| Bucket | Grain | Cardinality | Needs cross-user query? |
|---|---|---|---|
| **A. Matrix cells** (event × method on/off) | per user × event × method | high | only for digests (future) |
| **B. Per-event scalars** (interval, sound_key) | per user × event | medium | no |
| **C. Global modifiers** (quiet hours, DND, sounds, volume) | per user | one row | no (read with the user) |

The insight: **A and B share the same key `(user, event)`.** So they can collapse into *one* row per
event, with the methods + scalars as a small JSON payload. And C reads together with the user, so it
can sit in `users.settings`. That dissolves the table count fast. Here are the viable combinations,
loosest to tightest:

**Combo 1 — Zero new tables (pure JSON).**
Everything (A+B+C) in `users.settings.notifications` (the §3.1 document).
- *Tables added: 0.* Simplest possible; this is the JSON-first starting point (§3.8).
- *Cons:* no cross-user query (digests do a full scan). Fine until a digest feature exists.

**Combo 2 — One table, row per `(user, event)`, JSON payload; globals in `users.settings`. ◀ recommended graduation target**
```sql
CREATE TABLE notification_preferences (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    event       VARCHAR(64) NOT NULL,
    preferences JSON NOT NULL,   -- { "methods": {"email":true,...}, "interval":3, "sound":"chime" }
    created_at  TIMESTAMP NULL,
    updated_at  TIMESTAMP NULL,
    UNIQUE KEY uniq_user_event (user_id, event),
    KEY idx_event (event),       -- narrow digest scans to one event before JSON extraction
    CONSTRAINT fk_np_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);
```
A (cells) + B (interval/sound) live in the JSON `preferences` per event; C (globals) stays in
`users.settings.notifications.global`.
- *Tables added: 1.* This is essentially your proposal ("per-event setting in `notification_preferences` as JSON, globals in users JSON").
- Digest query: `WHERE event='mention'` narrows to a few rows per account, then
  `JSON_EXTRACT(preferences,'$.methods.email')` — and if one query gets hot you add a generated
  column + index on just that path. Good enough and far simpler than full normalization.
- *Cons:* method-level filtering is JSON extraction, not a plain indexed column. Acceptable at your
  volumes.

**Combo 3 — One fully-normalized table, row per `(user, event, method)`; globals in `users.settings`.**
The original §4.1 `notification_preferences` (cell-grain) **plus** stuffing interval/sound somewhere.
- Problem: B (interval/sound) is per-event, not per-cell, so it has nowhere clean to live → you
  either add a second table (back to two) or denormalize it onto every method row. This is the
  combination that *forced* the multi-table split. **Avoid unless you genuinely need SQL-level
  per-method queries.**

**Combo 4 — Two tables: `(user, event)` JSON table + a `notification_user_settings` table for globals.**
Like Combo 2 but globals get their own table instead of `users.settings`.
- Worth it only if you need to **query globals across users** at send time (e.g. a worker that skips
  everyone currently in quiet hours via SQL). Until then, globals-in-`users.settings` (Combo 2) is
  one fewer table for identical behavior.

#### Recommendation

| Stage | Layout | Tables |
|---|---|---|
| **Now** (start) | Combo 1 — all JSON | **0** |
| **When digests/cross-user queries land** | Combo 2 — one `(user,event)` JSON-payload table + globals in `users.settings` | **1** |
| Only if you later need SQL per-method or per-global queries | Combo 3 / Combo 4 | 2+ |

So the honest answer to "do we need the other two tables?": **no.** Go **Combo 1 → Combo 2**. You
land on **a single notifications table**, with globals in the user's JSON — exactly the combination
you sketched. The maximally-normalized §4.1–4.3 design stays in the doc only as the "if you ever
truly need cell-level SQL" escape hatch (Combo 3), not the target.

---

## 5. Catalog (code-owned schema)

The catalog is the contract. Example (`config/notifications.php` or PHP enums):

```php
return [
    'methods' => [   // delivery methods (NOT conversation channels)
        'desktop' => ['label' => 'Desktop', 'realtime' => true],
        'mobile'  => ['label' => 'Mobile',  'realtime' => true],   // push
        'browser' => ['label' => 'Browser', 'realtime' => true],   // Web Push / in-app
        'email'   => ['label' => 'Email',   'realtime' => false],
        'sound'   => ['label' => 'Sound',   'realtime' => true],
        // 'sip' is a softphone-connectivity concern, not a delivery method — see §8
    ],

    'groups' => ['conversations', 'mentions', 'sla', 'system'],

    'events' => [
        'conversation_started' => [
            'label'   => 'Conversation created',
            'group'   => 'conversations',
            'methods' => ['email' => true,  'desktop' => false, 'mobile' => false,
                          'browser' => false, 'sound' => true],
            'interval' => false,
        ],
        'conversation_assigned' => [
            'label'   => 'Conversation assigned to you',
            'group'   => 'conversations',
            'methods' => ['email' => false, 'desktop' => true, 'mobile' => true,
                          'browser' => true, 'sound' => true],
            'interval' => false,
        ],
        'active_idle_conversation' => [
            'label'   => 'Active idle conversation',
            'group'   => 'conversations',
            'methods' => ['browser' => true, 'desktop' => false, 'mobile' => false,
                          'email' => false, 'sound' => true],
            'interval' => ['min' => 1, 'max' => 10, 'default' => 1, 'unit' => 'minutes'],
        ],
        'mention' => [ /* ... */ ],
        'sla_warning' => [ /* ... */ ],
        'missed_conversation' => [ /* ... */ ],
        'conversation_transferred' => [ /* ... */ ],
    ],
];
```

The value in `methods` is the **default** for that cell. `interval => false` means the event has no
interval; otherwise it's the bounds + default.

This single source feeds: API `GET` (catalog + applicable methods + defaults), validation
(methods/events/intervals must exist in catalog), the resolver, and the frontend renderer (no
hardcoded rows).

---

## 6. API Design

Two resources: the **catalog/schema** (what's possible) and the **user's preferences** (their
choices). The frontend overlays the latter on the former. Mirror the existing `/v1/user/...`
namespace.

### 6.1 Read — `GET /v1/user/notification-preferences`

Returns catalog + resolved effective values, so the UI renders without guessing:

```json
{
  "methods": [
    { "key": "desktop", "label": "Desktop" },
    { "key": "mobile",  "label": "Mobile" },
    { "key": "email",   "label": "Email" },
    { "key": "browser", "label": "Browser" },
    { "key": "sound",   "label": "Sound" }
  ],
  "global": {
    "sounds_enabled": true,
    "dnd_enabled": false,
    "quiet_hours": { "enabled": false, "start": "22:00", "end": "08:00",
                     "timezone": "Asia/Tbilisi" },
    "mute_until": null
  },
  "groups": [
    {
      "key": "conversations",
      "label": "Conversations",
      "events": [
        {
          "key": "conversation_started",
          "label": "Conversation created",
          "methods": {
            "email":   { "applicable": true,  "enabled": true,  "source": "default" },
            "desktop": { "applicable": true,  "enabled": false, "source": "default" },
            "mobile":  { "applicable": true,  "enabled": false, "source": "default" },
            "browser": { "applicable": true,  "enabled": false, "source": "default" },
            "sound":   { "applicable": true,  "enabled": true,  "source": "user" }
          },
          "interval": null
        },
        {
          "key": "active_idle_conversation",
          "label": "Active idle conversation",
          "methods": {
            "browser": { "applicable": true, "enabled": true, "source": "user" },
            "email":   { "applicable": true, "enabled": false, "source": "default" }
            // ...
          },
          "interval": { "value": 1, "min": 1, "max": 10, "unit": "minutes" }
        }
      ]
    }
  ]
}
```

`source` (`default | user`) lets the UI show "(default)" hints and a "reset" affordance.
`applicable: false` → render the cell as `—` (not a checkbox), driven entirely by the catalog.

### 6.2 Update — `PATCH /v1/user/notification-preferences`

Accept **partial, sparse** updates (only what changed). Upsert per cell; no whole-blob overwrite.

```json
{
  "events": {
    "mention": { "methods": { "desktop": true, "email": false } },
    "active_idle_conversation": { "methods": { "browser": true }, "interval": 3 }
  },
  "global": {
    "sounds_enabled": false,
    "quiet_hours": { "enabled": true, "start": "22:00", "end": "08:00" }
  }
}
```

Server validates every `event`/`delivery_method` against the catalog, that the method is applicable
to the event, and interval bounds. Each provided cell → upsert (into the JSON document, or
`notification_preferences` once on tables); omitted cells are untouched. Setting a cell back to its
catalog default *may* drop the override (keeps storage sparse) — an implementation detail,
transparent to the client.

Optional convenience endpoints:
- `DELETE /v1/user/notification-preferences/{event}/{method}` → reset one cell to default.
- `POST   /v1/user/notification-preferences/reset` → wipe all overrides (back to defaults).
- `PATCH  /v1/user/notification-preferences/mute` → quick snooze (`mute_until`).

### 6.3 Backward compatibility

Keep `notification_sounds` and the legacy `settings.notifications` shape readable during migration
(see §12). `UserResource` continues to expose them until the frontend cuts over, then they're
dropped.

---

## 7. Frontend Page Structure & UX

### 7.1 Layout

The current single matrix is the right *primitive*; it just needs structure around it.

```
Notifications
├── Global
│   ├── [toggle] Notification sounds
│   ├── [toggle] Do not disturb (pause all)
│   └── Quiet hours [toggle] · 22:00 → 08:00 · timezone
│
├── Conversations                         ← group header
│   ┌──────────────────────────┬ Desktop · Mobile · Email · Browser · Sound ┐
│   │ Conversation created      │   ☐        ☐       ☑       ☐       ☑       │
│   │ Conversation assigned     │   ☑        ☑       ☐       ☑       ☑       │
│   │ Conversation transferred  │   ☑        ☐       ☐       ☑       ☑       │
│   │ Active idle conversation  │   —        —       ☐       ☑       ☑   [3▾]│  ← interval
│   │ Missed conversation       │   ☑        ☑       ☑       ☐       ☑       │
│   └──────────────────────────┴─────────────────────────────────────────────┘
│
├── Mentions
│   │ You were mentioned        │   ☑        ☑       ☑       ☑       ☑       │
│
└── SLA
    │ SLA warning               │   ☑        ☑       ☑       ☐       ☑       │
```

### 7.2 UX recommendations

- **Render from the catalog, not hardcoded rows.** The page maps over `groups → events → methods`
  from the API. New events appear automatically.
- **Group events into sections** (Conversations / Mentions / SLA / System). Matches how Slack,
  Teams, GitHub lay it out and stops the flat list from becoming a wall of checkboxes.
- **Column = delivery method, row = event.** Keep the matrix. Add column header icons + tooltips
  (already present in the screenshot).
- **Inapplicable cells render as `—`,** not an unchecked box, so users don't think a method is
  "off" when it's actually unavailable for that event.
- **Interval lives in the cell** for events that support it (the `[3▾]` next to Browser on idle),
  enabled only when that method is on.
- **Default hints + reset.** Show subtle "(default)" styling for non-overridden cells and a "Reset
  to defaults" action per group / globally.
- **Autosave per change** (debounced PATCH of just the changed cell) instead of one big "Save
  notifications" button — fewer lost edits, no whole-form clobber. If keeping an explicit Save,
  send only the diff.
- **Global controls on top:** sounds, DND, quiet hours, plus a quick "Mute for 1h / until tomorrow"
  snooze (Slack/Teams pattern).
- **Bulk affordances:** "turn a whole column on/off" (e.g. disable all email) and per-row
  "all/none" — large apps all have this.
- **Optimistic UI** with rollback on PATCH failure.
- **Avoid the word "channel" in UI copy.** Use "delivery method" or just the method names — "channel"
  in LiveCaller means a conversation source, and reusing it here would confuse agents.

---

## 8. Notification Delivery Methods

| Delivery method | Transport (today / proposed) | Notes |
|---|---|---|
| **Browser** | Laravel `broadcast` (WebSocket) → Web Push API | In-app + Web Push for background tabs. The screenshot's "Browser" maps here. |
| **Email** | Laravel `mail` | Already wired (`ConversationStarted::toMail`). Best for digests/quiet-hours fallback. |
| **Sound** | Client-side, gated by `sounds_enabled` + per-event `sound` cell | Promote from global boolean to a real per-event delivery method. Plays alongside an in-app/browser notification. |
| **Desktop** | Electron/desktop app push or Web Push (OS-level) | The desktop client subscribes; dispatch targets it. Needs a device/token registry. |
| **Mobile / Push** | FCM / APNs | Requires a `device_tokens` table (user_id, platform, token, last_seen). New transport driver. |
| **SIP** | **Not a delivery method** | `sip.auto_register` is a *softphone connectivity* setting, not "notify me about event X." Keep it out of the matrix; leave it under `settings.sip` (or a dedicated SIP settings section). Listing it as a delivery method is a category error. |

**Framework note.** Each delivery method maps onto a Laravel *notification channel driver* — `mail`,
`broadcast`, `database`, plus new `webpush`, `fcm`/`apns`. "Channel" here is Laravel's own term for
its transport drivers; it is **not** our delivery-method concept and **not** a conversation channel.
A **single delivery-method registry** replaces the `switch` in `NotificationService` — map
`method key → Laravel driver + payload builder` once.

Desktop/Mobile push require a **device token registry** (`device_tokens` table) so a method can be
"applicable" only when the user has a registered device. The catalog marks `applicable` accordingly.

### 8.1 Sound is special — it carries *which* sound, not just on/off

Every other delivery method is a pure boolean: email either sends or it doesn't. **Sound has a
second dimension — the audio that plays** — and users reasonably want *different* sounds per event
(a soft ding for a new conversation, a distinct chime for a mention, an urgent tone for an SLA
breach). So "sound" is modeled as **two things**:

1. **`sound` as a delivery-method cell** (boolean) — "play *a* sound for this event at all." This is
   the checkbox in the matrix, exactly like browser/email.
2. **A per-event sound *selection*** — "*which* sound for this event." This is a per-event scalar,
   structurally identical to `interval` (see §10 / the previous question): it lives in the per-event
   settings region, not on the per-method cell. You don't want a separate sound for browser vs
   desktop — the sound belongs to the *event*, played by whichever real-time method fires.

Plus two **global** sound settings (defaults in config, value per user — §12.2): a master
`sounds_enabled` switch and an optional `volume`.

**Catalog (config) — register the available sounds + each event's default:**

```php
// config/notifications.php
'sounds' => [                                  // the library of selectable sounds
    'ding'   => ['label' => 'Ding',   'file' => 'sounds/ding.mp3'],
    'chime'  => ['label' => 'Chime',  'file' => 'sounds/chime.mp3'],
    'knock'  => ['label' => 'Knock',  'file' => 'sounds/knock.mp3'],
    'alert'  => ['label' => 'Alert',  'file' => 'sounds/alert.mp3'],
],

'events' => [
    'conversation_started' => [
        'methods'       => ['sound' => true, /* ... */],
        'sound_default' => 'ding',             // played unless the user overrides
    ],
    'mention' => [
        'methods'       => ['sound' => true, /* ... */],
        'sound_default' => 'chime',
    ],
    'sla_warning' => [
        'methods'       => ['sound' => true, /* ... */],
        'sound_default' => 'alert',
    ],
],
```

**Per-user override (JSON) — `sound` sits beside `interval`, both per-event scalars:**

```json
"events": {
  "mention": {
    "methods": { "sound": true },
    "sound": "knock"          // user picked a different sound; absent → catalog sound_default
  },
  "active_idle_conversation": {
    "methods": { "sound": true },
    "interval": 3
    // no "sound" key → uses the event's sound_default
  }
}
```

**Table target (§4.2)** — add one column to `notification_event_settings`, since the sound choice is
the same shape as the interval (one row per user × event):

```sql
ALTER TABLE notification_event_settings
    ADD COLUMN sound_key VARCHAR(32) NULL;   -- references a key in config 'sounds'; NULL = default
```

**Resolution at send time** (client-side, since sound is played in the browser/desktop app):

```
play sound IF:
    global sounds_enabled (userOverride ?? config default)         -- master mute
    AND effective(user, event, 'sound') == true                    -- the matrix cell
THEN:
    soundKey = userOverride[event].sound ?? config.events[event].sound_default
    file     = config.sounds[soundKey].file
    volume   = userOverride.volume ?? config default
```

**API** — `GET` exposes the sound library + each event's resolved sound so the UI can render a
picker; `PATCH` accepts a `sound` key per event (validated against `config.sounds`):

```json
// GET adds, per event:
"sound": { "enabled": true, "selected": "knock", "default": "chime",
           "options": [ {"key":"ding","label":"Ding"}, {"key":"chime","label":"Chime"}, ... ] }

// PATCH:
{ "events": { "mention": { "methods": { "sound": true }, "sound": "knock" } },
  "global": { "sounds_enabled": true, "volume": 0.7 } }
```

**UX:** in the matrix, the Sound cell is a checkbox **plus** a small sound-picker dropdown (with a
▶ preview button) that's enabled only when the cell is on — structurally the same pattern as the
interval `[3▾]` control on the idle row. Global `sounds_enabled` + volume live in the top "Global"
block. This mirrors how Slack/Discord do it: a master sound toggle, plus a per-event sound chooser.

> **Why not a sound per delivery method (browser-sound vs desktop-sound)?** Because only one
> real-time surface actually fires for a given event in practice, and users think "this *event*
> sounds like X," not "this event sounds like X in the browser but Y on desktop." Keeping sound
> selection per-event (not per-method) matches the mental model and avoids a combinatorial mess.

### 8.2 Adding external delivery methods (Telegram, Slack, WhatsApp, push, …)

The catalog + registry design exists precisely so new delivery methods are cheap to add. Adding one
is **not** a schema change to the preference storage — it's a config entry plus a driver. Recipe:

1. **Catalog entry** — add the method to `config/notifications.php` and set its per-event defaults /
   applicability. The frontend renders the new column automatically (it maps over the catalog).
   ```php
   'methods' => [
       // ...existing...
       'telegram' => ['label' => 'Telegram', 'realtime' => true, 'requires_connection' => true],
       'slack'    => ['label' => 'Slack',    'realtime' => true, 'requires_connection' => true],
   ],
   ```
2. **Registry mapping + driver** — map the method key to a Laravel notification channel driver and a
   payload builder (the registry that replaced the old `switch`). Install the package
   (e.g. `laravel-notification-channels/telegram`; Slack ships a `slack` channel).
   ```php
   'telegram' => ['driver' => TelegramChannel::class, 'payload' => 'toTelegram'],
   ```
3. **Payload builder** — add `toTelegram($notifiable)` / `toSlack($notifiable)` to the notification
   classes (or one generic formatter that renders title + body + link per method).
4. **(Only for external-account methods) connection + destination registry** — see below.

That's it. No change to how preferences are stored; the matrix just gains a column.

#### The one new concept: a destination registry

Email uses `user.email`; browser uses the live WebSocket session; sound plays locally. These have an
**implicit address**. Telegram, Slack, WhatsApp, and mobile/desktop **push** do not — you must know
*where* to send (a Telegram `chat_id`, a Slack user ID / OAuth token, a device push token). So these
methods need a per-user **destination registry** — which **generalizes** the `device_tokens` table
mentioned in §8 (push is just one kind of external destination):

```sql
CREATE TABLE notification_destinations (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT UNSIGNED NOT NULL,
    method     VARCHAR(32)  NOT NULL,   -- 'telegram','slack','whatsapp','mobile','desktop','webpush'
    address    VARCHAR(255) NOT NULL,   -- chat_id / slack user id / device token / endpoint
    meta       JSON NULL,               -- platform, scopes, token expiry, display label
    status     ENUM('active','revoked') NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY uniq_user_method_address (user_id, method, address),
    KEY idx_user_method (user_id, method),
    CONSTRAINT fk_nd_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);
```

Laravel routes to it via `routeNotificationForTelegram()` / `routeNotificationForSlack()` on the
`User`, which look up the active `address` for that method.

> Note: this is the *one* table per-device/per-account methods genuinely need — but it's a **routing
> registry, not a preference**. It's orthogonal to the matrix (§4.6): preferences stay per-user;
> destinations are per-connected-account. Don't conflate them.

#### Connection flow + applicability gating

A method flagged `requires_connection: true` is only usable once the user **links** their account:

- **Connect UX:** "Connect Telegram" → bot deep-link with a one-time token the bot echoes back, so
  you capture their `chat_id`. "Connect Slack" → OAuth. On success, write a row to
  `notification_destinations`.
- **Applicability:** the resolver/API reports `applicable: false` (render a **"Connect"** button
  instead of a checkbox) until an active destination exists — the same rule as push being unavailable
  with no registered device. Once connected, the column behaves like any other method.
- **Revocation:** if the token dies (user blocks the bot, revokes OAuth), mark the destination
  `revoked`; the method silently reverts to "Connect" and dispatch skips it.

#### What does *not* change

The resolver formula, the JSON/Combo-2 preference storage, the catalog-driven frontend, defaults in
config, and the sound model are all untouched. Adding Telegram is: **catalog row + registry row +
payload method + (for external accounts) a destination row & connect flow.** No migration to the
preference store, no frontend rewrite.

> **Team/shared-channel posting** (e.g. "post all new conversations to a shared `#support` Slack
> channel") is a *different* feature — that's an account-level integration/routing destination, not a
> per-agent preference. It would hang off `accounts` (a shared destination), not the per-user matrix.
> Out of scope here, but worth not confusing with personal Telegram/Slack delivery.

---

## 9. Notification Event Types

Catalog-defined. Initial set (extend freely — adding one is a config edit, not a migration):

| Event key | Label | Group | Typical default methods | Interval |
|---|---|---|---|---|
| `conversation_started` | Conversation created | Conversations | email | — |
| `conversation_assigned` | Conversation assigned to you | Conversations | desktop, mobile, browser, sound | — |
| `conversation_transferred` | Conversation transferred to you | Conversations | desktop, browser, sound | — |
| `active_idle_conversation` | Active idle conversation | Conversations | browser, sound | ✅ 1–10 min |
| `missed_conversation` | Missed conversation | Conversations | email, desktop, sound | — |
| `mention` | You were mentioned | Mentions | desktop, mobile, browser, email, sound | — |
| `sla_warning` | SLA warning / breach approaching | SLA | desktop, mobile, email, sound | (threshold) |
| `callback_requested` | Callback requested | Conversations | desktop, sound | — |
| *(system)* `weekly_report` | Weekly report ready | System | email | — |

Each event declares its **applicable delivery methods** and **per-method defaults** in the catalog
(§5), so not every event exposes every method (e.g. a weekly report has no "sound").

---

## 10. Intervals, Quiet Hours, Per-Method, Global, Extensibility

- **Per-method settings:** the core of the design — every cell is independent (one entry per
  event × delivery method).
- **Intervals:** per-event scalar (`notification_event_settings.interval_value` once on tables;
  `events.{event}.interval` in JSON), bounds from the catalog. Used by events like
  `active_idle_conversation` (re-notify every N minutes) and potentially SLA thresholds.
- **Sound selection:** another per-event scalar, alongside interval (`sound_key` /
  `events.{event}.sound`) — *which* sound plays for the event, chosen from the catalog `sounds`
  library, with a master `sounds_enabled` + `volume` global. Full design in §8.1.
- **Quiet hours:** global per-user (start/end/timezone). At send time, real-time methods
  (desktop/mobile/browser/sound) are suppressed during quiet hours; email may still queue or convert
  to a digest. Handles midnight-wrap (start > end).
- **Global switches:** `dnd_enabled` (pause all), `sounds_enabled` (master mute for the sound
  method), `mute_until` (temporary snooze). Evaluated as a gate *after* the matrix.
- **`is_away` interaction:** optionally suppress real-time methods while away (configurable). Keep
  `is_away` itself where it is — it's a presence flag, not a notification pref.
- **Extensibility:** add an event or delivery method by editing the **catalog** — no migration, no
  API change, no frontend change (the page renders whatever the catalog returns). New *transport*
  (e.g. APNs) = one new Laravel driver + registry entry.
- **Digests (future):** a normalized table makes "collect all `email` prefs for event X" a single
  indexed query — a digest worker becomes trivial. Hard with the JSON blob (full scan); this is the
  signal to graduate to tables (§3.8).

---

## 11. How Large Companies Do It

Common patterns across Slack / Discord / Teams / Intercom / GitHub / Linear:

- **Event × delivery-method matrix** as the core model — exactly the grid in the screenshot, but
  with grouping and many events. (These products call the columns "channels"; we say "delivery
  methods" to avoid colliding with our conversation channels.)
- **Server-defined catalog** of notification *types* and *delivery methods*; clients render from it
  rather than hardcoding. New types ship without a client release.
- **Layered resolution:** user preference → system default, most-specific wins. (Large orgs add a
  workspace/admin policy layer on top; we're intentionally skipping that for now.)
- **Normalized storage** for scale, not a per-user JSON blob — because background workers (digests,
  push fan-out) must *query* preferences efficiently across millions of users. (At smaller scale,
  structured JSON is a legitimate starting point — see §3.)
- **Global modifiers layered on top:** Do-Not-Disturb / quiet hours / snooze ("notification
  schedule" in Slack, "quiet hours" in Teams, "snooze" everywhere).
- **Defaults that are sensible and overridable**, with a visible "reset to default" and an
  indication of which settings are non-default.
- **Digest/batching** as a first-class delivery mode (email "every 15 min / hourly / daily"),
  enabled by queryable storage.
- **Device/token registry** for push, with methods marked unavailable when no device is registered.

Our current design has the *grid* but none of: catalog, layering, structured storage, defaults,
quiet hours, or actually honoring the prefs at dispatch.

---

## 12. Final Recommendation

### 12.0 Bottom line

Build a **catalog-driven, JSON-first, table-ready** notification system:

1. **Catalog in `config/notifications.php`** is the source of truth — defines events, delivery
   methods, applicability, intervals, sounds, groups, and **all defaults** (including global
   defaults like `sounds_enabled` / quiet-hours). Add an event or method by editing config — no
   migration, no frontend change.
2. **A storage-agnostic `NotificationPreferenceResolver`** computes
   `effective = userOverride ?? catalogDefault`, then applies global modifiers (quiet hours / DND /
   mute). It's the single seam everything reads through.
3. **A delivery-method registry** replaces the hardcoded `switch` in `NotificationService`, which is
   made to honor **every** event — fixing the live bug where `active_idle_conversation` toggles do
   nothing.
4. **Storage = structured JSON in `users.settings.notifications`** (Combo 1, *zero new tables*),
   storing only sparse overrides. Globals live in the `…global` block of that same doc. Graduate to
   **one** `(user, event)` JSON-payload table (Combo 2) *only if* digests/cross-user queries ever
   land — and even then, never one row per delivery method.
5. **Sound** = an on/off cell in the matrix **plus** a per-event sound selection (like `interval`),
   under a global `sounds_enabled` + `volume`.
6. **Naming:** call the columns **delivery methods**, never "channel" (reserved for conversation
   channels). **SIP stays out** — it's softphone connectivity, not a delivery method.
7. **Out of scope for now:** account-level policy, push/mobile, email digests, audit. The
   architecture leaves room for each but builds none of them yet.
8. **Leave `sip` and `is_away` in `users.settings`** — they're not notifications.

**Do first (highest value, ~no migration): Phase 1** — catalog + resolver + registry + dispatch-bug
fix. That alone fixes problems #2–#5 and #9 from §1.

### 12.1 How to approach this — weighing the options

Before the rollout, the real question is *which shape of solution to commit to*. Three are on the
table; the phased plan below is the consequence of picking the third.

**Option A — Minimal patch.** Keep the current JSON shape, just fix the dispatch bug (make
`NotificationService` read every event's prefs) and split `sip`/`is_away` out of the blob.
- *Pros:* a day of work; no API or frontend change; immediately stops the "toggle does nothing" bug.
- *Cons:* still no catalog, so adding an event is still a 3-file change; no defaults model (new
  users still get nothing); no quiet hours/intervals/grouping. You'll redo this work the moment the
  feature grows.
- *Verdict:* correct as a hotfix, insufficient as the redesign you actually asked for.

**Option B — Full normalized tables now (big bang).** Build all three tables, migrate, rewrite the
API and frontend in one push.
- *Pros:* "done properly" in one go; queryable from day one.
- *Cons:* pays the full migration + concurrency + assembly cost up front for benefits (digests,
  cross-user queries) you don't have a feature for yet (§3.6); largest blast radius; highest risk of
  a stalled half-migrated state.
- *Verdict:* over-investment for LiveCaller's current scale. Right destination, wrong time.

**Option C — Catalog-driven, JSON-first, table-ready (recommended).** Put all the *intelligence* in
a code-owned catalog + a storage-agnostic resolver, store user overrides as structured JSON now, and
keep the door open to tables behind the resolver.
- *Pros:* fixes every concrete problem in §1 with **zero migrations**; the catalog gives defaults +
  extensibility immediately; the resolver is the single seam, so the eventual JSON→tables move is
  localized, not a rewrite; you only pay for tables when a querying feature (digests) actually lands.
- *Cons:* slightly more up-front structure than Option A (a catalog file + a resolver service); cross-
  user queries remain inefficient until you graduate (acceptable — you have no such query today).
- *Verdict:* best ratio of value to risk. **This is the recommendation.**

The decision hinges on one fact: **the only thing tables buy you that JSON can't is efficient
cross-user querying, and you have no feature that needs it yet.** So invest in the part that pays off
regardless of storage (the catalog + resolver), and defer the part that only pays off later (tables).

### 12.2 The organizing principle: defaults in config, overrides per user

This is the backbone of Option C and directly answers "can global settings live in the config file?"
— **yes.** Cleanly separate the two:

| | Lives in | Cardinality | Changes by |
|---|---|---|---|
| **System defaults** — default matrix (which event×method is on), default quiet-hours/DND/sound values, applicability, interval bounds, labels, groups | **`config/notifications.php`** (the catalog) | one, shared by everyone | a deploy |
| **Per-user overrides** — only the cells/values this user actually changed | per-user storage (JSON now, tables later) | sparse, 0–N per user | a `PATCH` from the user |

Consequences of this split (all good):

- **A brand-new user stores nothing.** Every value resolves from config. No seeding, no backfill for
  defaults, no rows for the 90% of users who never open the page. (Directly fixes problem #5 — the
  "`settings = null` → everything off" bug.)
- **You can re-tune product defaults with a deploy**, and it retroactively applies to every user who
  hasn't overridden that cell — usually exactly what you want for "we should notify on mentions by
  default now."
  - *One caveat to be aware of:* because non-overridden cells follow the config, changing a default
    *does* change behavior for those users. That's the intended semantics, but note it so a default
    flip is a conscious decision, not a surprise. If you ever need a default change to **not** affect
    existing users, you'd snapshot the old default into their overrides at migration time.
- **The resolver formula stays trivial and storage-agnostic:**
  `effective(user, event, method) = userOverride[event][method] ?? config.events[event].methods[method]`
  …then gate by the user's global modifiers (quiet hours / DND / mute), whose *defaults* also come
  from config but whose *values* are per-user.

Note this means even the *global modifiers* (quiet hours, DND, sounds) follow the same rule: their
**default** (`sounds_enabled: true`, `quiet_hours.enabled: false`) sits in config; only a user who
changes them stores a value. So "global settings in config + per-user settings separate" isn't a
third storage location — it's the same defaults-vs-overrides split applied to the global block too.

### 12.3 Rollout

Given Option C, a rollout that avoids a big-bang rewrite:

**Phase 1 — Backend foundations (no UI change, no migration)**
1. Add `config/notifications.php` catalog (events, delivery methods, defaults, intervals, groups).
2. Add a `NotificationPreferenceResolver` service:
   `effective(user, event, method)` = `userOverride ?? catalogDefault`, gated by global modifiers.
   Make it **storage-agnostic** (reads from JSON today, tables later).
3. Replace the hardcoded `switch` in `NotificationService` with a **delivery-method registry** driven
   by the catalog, and make `process()` consult the resolver for **every** event — fixing the bug
   where `active_idle_conversation` prefs are stored but never honored.

**Phase 2 — Clean JSON + API**
4. Restructure `settings.notifications` to the §3.1 shape (versioned, sparse overrides), validated
   against the catalog. One-time in-place migration of existing blobs.
5. Ship `GET/PATCH /v1/user/notification-preferences` (catalog + resolved values, sparse updates).
6. Keep `PATCH /v1/user/notifications/settings` and `UserResource.settings.notifications` working
   (deprecated) until the frontend cuts over.

**Phase 3 — Frontend**
7. Rebuild the page to render from the catalog: grouped sections, per-method cells, interval in
   cell, global controls (sounds, DND, quiet hours), autosave-diff, default hints + reset.

**Phase 4 — Cleanup**
8. Drop the legacy `settings.notification_sounds` flat flag and the old endpoint once unused. Leave
   `sip` and `is_away` in `settings` (they are not notifications).

**Phase 5 — Graduate to ONE table (only when a querying feature lands)**
9. Add the single `notification_preferences` table — **Combo 2** (§4.6): one row per `(user, event)`
   with a JSON `preferences` payload (methods + interval + sound). Keep globals in
   `users.settings.notifications.global`. **Backfill** from the JSON document; **dual-write** for one
   release. (Do *not* build the three-table design unless you later need cell-level SQL — §4.6 Combo 3.)
10. Flip the resolver's read source from JSON to the table. Because the resolver is the only reader,
    this is localized. Drop dual-write afterward.

**Phase 6 — Extend (as needed)**
11. Add events (assigned, transferred, mention, SLA, missed) to the catalog and wire their dispatch
    through the resolver.
12. Add push delivery methods (`webpush`, `fcm`/`apns`) + a `device_tokens` registry.
13. Add quiet-hours enforcement and (optionally) email digests — the trigger for Phase 5.

### What to keep vs. move

- **Restructure (and later move to tables):** everything under `notifications.*` and
  `notification_sounds`.
- **Keep in `settings` JSON:** `sip.auto_register`, `is_away` — low-cardinality, read-as-a-unit,
  never queried, unrelated to notifications. Don't pollute the new system with them.

### Net result

A clean separation: a **code-owned catalog** defines what's possible, a **storage-agnostic resolver +
delivery-method registry** make dispatch honor every preference, **structured JSON** holds user
choices with proper defaults today (table-ready for the day queries demand it), and a
**catalog-driven frontend** that grows without code changes — plus quiet hours, intervals, per-method
control, and a clear path to push/digests. No more unrelated settings sharing one JSON junk drawer,
no toggles that silently do nothing, and no "channel" name clashing with our conversation channels.

---

## 13. Finalized model (current canonical decision)

> This section supersedes the exploration in §3–§4 for the actual build. It reflects the decisions we
> converged on: **user (agent) notifications only**, **sound is a per-event attribute (not a delivery
> method)**, delivery methods are **email / browser / desktop / mobile**, and the event set is
> **opened / assigned / idle / missed**. Globals (quiet hours, DND, volume) are deferred.

### 13.1 The catalog — `config/user-notifications.php` (code, the contract)

```php
return [
    // delivery methods = real destinations (the matrix columns)
    'methods' => [
        'email'   => ['label' => 'Email'],
        'browser' => ['label' => 'Browser'],
        'desktop' => ['label' => 'Desktop'],
        'mobile'  => ['label' => 'Mobile'],   // push to the mobile app
    ],

    // selectable sounds (sound is an ATTRIBUTE of an event, not a method)
    'sounds' => [
        'ding'  => ['label' => 'Ding',  'file' => 'sounds/ding.mp3'],
        'chime' => ['label' => 'Chime', 'file' => 'sounds/chime.mp3'],
        'alert' => ['label' => 'Alert', 'file' => 'sounds/alert.mp3'],
    ],

    // user (agent) notification events; method values = DEFAULT on/off per cell
    'events' => [
        'conversation_opened' => [
            'label'   => 'Conversation opened',
            'methods' => ['email' => true, 'browser' => true, 'desktop' => false, 'mobile' => false],
            'is_muted' => true,
            'sound'   => 'ding',
        ],
        'conversation_assigned' => [
            'label'   => 'Conversation assigned to you',
            'methods' => ['email' => false, 'browser' => true, 'desktop' => true, 'mobile' => true],
            'sound'   => 'chime',
        ],
        'active_idle_conversation' => [
            'label'    => 'Active idle conversation',
            'methods'  => ['email' => false, 'browser' => true, 'desktop' => false, 'mobile' => false],
            'sound'    => 'ding',
            'interval' => ['min' => 1, 'max' => 10, 'default' => 1],
        ],
        'conversation_missed' => [
            'label'   => 'Conversation missed',
            'methods' => ['email' => true, 'browser' => true, 'desktop' => true, 'mobile' => false],
            'sound'   => 'alert',
        ],
    ],
];
```

Notes tied to the conversation lifecycle (`opened → assigned to chatbot/ai/user → … → missed`):
- **`conversation_assigned`** notifies only when the assignment target is a **user (operator)** —
  assignments to `chatbot`/`ai` don't ping a human.
- **`sound`** is a sibling of `methods` (a per-event attribute); `null` = the notification still
  fires per `methods`, just silent. It's played only when a real-time method (browser/desktop)
  actually shows the notification.

### 13.2 Storage option A — reuse the existing `users.settings` JSON column (recommended start)

No new column needed: `users.settings` is **already** a `json` column with the `HasSettings` trait
(the `setting()` accessor). Store the prefs under a clean, namespaced `notifications` key:

```php
// nothing to migrate — settings is already json. Just write under settings.notifications:
$user->setting(['notifications' => [...]])->save();
```

Stores **only overrides**; everything else falls back to the catalog. The `notifications` key sits
alongside the existing `sip` / `is_away` keys, each its own namespace (not the old flat mush):

```json
// users.settings
{
  "sip": { "auto_register": true },
  "is_away": false,
  "notifications": {
    "version": 2,
    "events": {
      "conversation_assigned":    { "methods": { "mobile": false } },
      "active_idle_conversation": { "methods": { "browser": true }, "interval": 3, "sound": null }
    }
  }
}
```

Resolution via the existing accessor:

```php
$on       = $user->setting("notifications.events.$e.methods.$m") ?? config("user-notifications.events.$e.methods.$m");
$sound    = $user->setting("notifications.events.$e.sound")      ?? config("user-notifications.events.$e.sound");
$interval = $user->setting("notifications.events.$e.interval")   ?? config("user-notifications.events.$e.interval.default");
```

Tables **and** migrations added: **0**. Adding an event/method/sound is a config edit only.

> Doesn't this re-create the §1 "junk drawer"? No. Problem #1 was the *old flat, unstructured,
> partially-honored* mix — not the column itself. A **versioned, catalog-validated, namespaced
> `settings.notifications` sub-document** is clean. `sip` and `is_away` stay as their own sibling
> keys. Reusing the column is the pragmatic win: the storage already exists.

### 13.3 Storage option B — a separate table (the vision)

Because the methods are now a **fixed, small set** (email/browser/desktop/mobile) and sound/interval
are per-event scalars, one row per `(user, event)` maps cleanly to **explicit columns** — no JSON
needed, fully queryable:

```sql
CREATE TABLE user_notification_preferences (
    id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id   BIGINT UNSIGNED NOT NULL,
    event     VARCHAR(64) NOT NULL,        -- 'conversation_opened', 'conversation_assigned', ...

    -- one nullable column per delivery method. NULL = inherit the catalog default.
    email     TINYINT(1) NULL,
    browser   TINYINT(1) NULL,
    desktop   TINYINT(1) NULL,
    mobile    TINYINT(1) NULL,

    -- per-event attributes
    sound            VARCHAR(32) NULL,      -- sound key, '' = silent, NULL = inherit default
    interval_minutes SMALLINT UNSIGNED NULL,-- only events that support it; NULL = inherit default

    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,

    UNIQUE KEY uniq_user_event (user_id, event),
    KEY idx_event (event),                  -- "who wants X on event Y" lookups
    CONSTRAINT fk_unp_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);
```

Semantics — identical resolution to JSON, but in columns:

- One row per `(user, event)` the user customized. **At most 4 rows per user** (4 events today).
- A `NULL` cell = "inherit the catalog default for that method/attribute" (preserves unset-vs-off).
- A user who never touched settings = **zero rows** = all defaults.
- Resolver: `value = row.column (if NOT NULL) ?? config default`.

Example rows for user 42 (turned mobile off for assigned; set idle browser-on, interval 3):

```
 user_id | event                    | email | browser | desktop | mobile | sound | interval_minutes
 --------+--------------------------+-------+---------+---------+--------+-------+-----------------
   42    | conversation_assigned    | NULL  | NULL    | NULL    |   0    | NULL  | NULL
   42    | active_idle_conversation | NULL  |   1     | NULL    | NULL   | NULL  |   3
```

**Pros over JSON**
- Fully **queryable** without JSON functions: `SELECT user_id FROM user_notification_preferences
  WHERE event='conversation_missed' AND email = 1` (the digest/fan-out case).
- **Type-safe columns** + constraints; clean to index.
- Still **sparse** (NULL = default) and still at most ~4 rows/user.

**Cons vs JSON**
- Adding a delivery method (e.g. `telegram`) = `ALTER TABLE … ADD COLUMN` (a migration), whereas JSON
  needs none. Acceptable while methods are a small, slow-changing set; if methods will grow a lot,
  prefer JSON or a `(user, event, method)` sub-table.
- `sound` / `interval_minutes` are sparse columns (NULL on most rows) — harmless, slightly untidy.

### 13.4 Which to use

|                              | `settings.notifications` JSON (A)       | Separate table (B)                                   |
| ---------------------------- | --------------------------------------- | ---------------------------------------------------- |
| Tables / migrations          | 0 (column already exists)               | 1 table; +migration to add a method                  |
| Cross-user queries (digests) | full scan / JSON-extract                | plain indexed `WHERE`                                |
| Adding an event              | config only                             | config only                                          |
| Adding a delivery method     | config only                             | config **+ ALTER TABLE**                             |
| Rows per user                | 0 (one column)                          | 0–4                                                  |
| Best when                    | methods may grow; no digest workers yet | method set is stable; you want SQL queries / digests |

**Recommendation:** start with **A (JSON column)** — zero migrations, fits the current scale. Move to
**B (separate table)** the day you build digests or cross-user "who wants X" workers. Either way the
**resolver is the only reader**, so switching A→B is a localized change, not a rewrite.

### 13.5 The one table that is *not* optional for push/external methods

`mobile` (and later Telegram/Slack) can't actually *send* without a device token / linked address.
That's a separate **routing registry** (`notification_destinations`, §8.2) — orthogonal to
preferences (A or B). Until a device is registered, the UI shows the `mobile` cell as "Connect/—";
the preference shape doesn't change, only sendability is gated.
