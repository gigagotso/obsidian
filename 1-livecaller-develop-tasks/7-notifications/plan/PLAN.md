# Notification Preferences — Implementation Plan (minimal)

Goal: a clean, catalog-driven user-notification system with **three layers**, reusing existing JSON
columns (no migrations to start). Minimal scope so we can review and start coding.

## Resolution model (3 components, low → high precedence)

```
1. config/user-notifications.php           → catalog defaults  (code)
2. accounts.settings.users.notifications   → account override  (admin-managed, account-wide)
3. users.settings.notifications            → user override     (per agent)

effective(user, event, method) =
    userOverride   ??           // the agent's own choice
    accountOverride ??          // account-wide default set by an admin
    catalogDefault              // from config

# optional (can skip v1): if account marks a cell "locked", account value wins and the
# user cannot override it. Add later; not required for first version.
```

One resolver service is the only reader of all three → swapping storage later stays localized.

---

## Component 1 — `config/user-notifications.php` (catalog)

Source of truth for methods, sounds, events, per-cell defaults, intervals. (Full version in
`../NOTIFICATION-PREFERENCES-ARCHITECTURE.md` §13.1.)

```php
return [
    'methods' => [ 'email'=>['label'=>'Email'], 'browser'=>['label'=>'Browser'],
                   'desktop'=>['label'=>'Desktop'], 'mobile'=>['label'=>'Mobile'] ],
    'sounds'  => [ 'ding'=>['file'=>'sounds/ding.mp3'], 'chime'=>['file'=>'sounds/chime.mp3'],
                   'alert'=>['file'=>'sounds/alert.mp3'] ],
    'events'  => [
        'conversation_opened'      => ['label'=>'Conversation opened',
            'methods'=>['email'=>true,'browser'=>true,'desktop'=>false,'mobile'=>false], 'sound'=>'ding'],
        'conversation_assigned'    => ['label'=>'Conversation assigned to you',
            'methods'=>['email'=>false,'browser'=>true,'desktop'=>true,'mobile'=>true], 'sound'=>'chime'],
        'active_idle_conversation' => ['label'=>'Active idle conversation',
            'methods'=>['email'=>false,'browser'=>true,'desktop'=>false,'mobile'=>false], 'sound'=>'ding',
            'interval'=>['min'=>1,'max'=>10,'default'=>1]],
        'conversation_missed'      => ['label'=>'Conversation missed',
            'methods'=>['email'=>true,'browser'=>true,'desktop'=>true,'mobile'=>false], 'sound'=>'alert'],
    ],
];
```

## Component 2 — Account override (`accounts.settings.users.notifications`)

Account admins set account-wide defaults. Stored in the existing `accounts.settings` JSON via the
`setting()` accessor, namespaced under `users` (these are account-wide defaults *for users*; keeps
top-level `accounts.settings.notifications` free for account-entity notifications). Sparse — only
overridden cells.

```json
// accounts.settings.users.notifications
{
  "events": {
    "conversation_missed": { "methods": { "email": true } }   // account default: email everyone on missed
  }
}
```

## Component 3 — User override (`users.settings.notifications`)

Each agent overrides their own. Stored in existing `users.settings` JSON. Sparse.

```json
// users.settings.notifications
{
  "events": {
    "conversation_assigned":    { "methods": { "mobile": false } },
    "active_idle_conversation": { "interval": 3, "sound": null }
  }
}
```

---

## Resolver service

`app/Services/NotificationPreferenceResolver.php`

```php
enabled(User $u, string $event, string $method): bool   // user ?? account ?? catalog
sound(User $u, string $event): ?string
interval(User $u, string $event): ?int
resolveAll(User $u): array     // full merged tree for API GET
```

Reads: `config('user-notifications')`, `$u->account->setting('users.notifications')`, `$u->setting('notifications')`.

---

## Permissions (Spatie, `resource:action`)

- New permission: **`notifications:manage`** (account-scoped) — controls editing the **account-level**
  override (Component 2). Per-user prefs (Component 3) need no special permission (any auth'd user
  edits their own).
- Seeder: `database/seeds/NotificationPermissionsSeeder.php` — create `notifications:manage`, assign
  to the `admin` role (operators don't get it).

```php
Permission::firstOrCreate(['name' => 'notifications:manage', 'guard_name' => 'api']);
Role::where('name','admin')->each(fn($r) =>
    $r->givePermissionTo('notifications:manage'));
```

---

## API (minimal) — under existing `auth:api` + `account.ownership` group in `routes/api.php`

### Account-level (admin) — gated by `can:notifications:manage`
```
GET   /v1/account/notifications     → catalog + current account overrides   (AccountNotificationsController@index)
PATCH /v1/account/notifications     → upsert account overrides (sparse)      (AccountNotificationsController@update)
```
`update` validates events/methods/sound/interval against the catalog, then
`$user->account->setting(['users.notifications' => ...])->save()`.

### User-level (any authenticated agent)
```
GET   /v1/user/notifications        → catalog + RESOLVED values for me (account+user merged, w/ source)
PATCH /v1/user/notifications        → upsert MY overrides (sparse)
```
`update` validates against catalog, then `$request->user()->setting(['notifications' => ...])->save()`.

> Replaces the current `PATCH /v1/user/notifications/settings` (`UserController@updateNotifications`)
> which is hardcoded to 2 events. Keep old route working until frontend cuts over, then remove.

### GET /v1/user/notifications — response shape (drives the UI)
```json
{
  "methods": [{"key":"email","label":"Email"}, ...],
  "events": [
    { "key":"conversation_assigned", "label":"Conversation assigned to you",
      "methods": {
        "email":  {"enabled":false,"source":"default"},
        "mobile": {"enabled":false,"source":"user"}
      },
      "sound": {"selected":"chime","options":["ding","chime","alert"]},
      "interval": null }
  ]
}
```
`source` = `default | account | user` (so UI can show where a value came from / reset).

---

## Dispatch fix (the live bug)

`app/Services/NotificationService.php`: replace the hardcoded `eventsAliasMap`/`switch` with a
**method registry** (`email→mail`, `browser/desktop→broadcast`, `mobile→push later`) and consult the
resolver for **every** event, so all configured events are honored (today only `conversation_started`
is).

---

## Build order (minimal, reviewable steps)

1. `config/user-notifications.php` (Component 1).
2. `NotificationPreferenceResolver` service (3-layer merge).
3. `NotificationPermissionsSeeder` → `notifications:manage` → admin role.
4. Account API: `AccountNotificationsController` (GET/PATCH) + routes (gated) → writes `accounts.settings.users.notifications`.
5. User API: `UserNotificationsController` (GET/PATCH) + routes → writes `users.settings.notifications`; deprecate old endpoint.
6. Wire `NotificationService` dispatch to the resolver (fix bug).
7. Frontend rebuild (separate task).

## Out of scope (for now)
Quiet hours / DND / volume, account-lock/force flag, push device-token registry (`mobile`/Telegram/
Slack actual sending), email digests, separate DB tables (stay on JSON until a querying feature needs them).
