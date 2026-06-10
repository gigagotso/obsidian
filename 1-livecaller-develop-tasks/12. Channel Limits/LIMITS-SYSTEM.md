# Limits System

How `config/limits.php` works, how the rate-limit "pool" fills and resets, and how the
new **per-account + per-channel_line conversation-creation** cap fits in.

---

## 1. The big picture

There are **three independent layers** of limiting in this app. They do not replace each
other — a request can be checked by all three.

```
                          ┌─────────────────────────────────────────────┐
                          │  LAYER 1 — Global HTTP throttle              │
                          │  240 req/min per route group (web / api)     │
                          │  Kernel.php:77,89   throttle:240,1,...        │
                          └─────────────────────────────────────────────┘
                                              │
                          ┌─────────────────────────────────────────────┐
                          │  LAYER 2 — Per-actor throttle                │
                          │  ONE visitor / ONE IP                        │
                          │  e.g. widget: throttle:5|...,conversation_…   │
                          └─────────────────────────────────────────────┘
                                              │
                          ┌─────────────────────────────────────────────┐
                          │  LAYER 3 — config/limits.php (this document) │
                          │  • messaging rate (requests_per_minute)      │
                          │  • upload / message validation               │
                          │  • account resource caps                     │
                          │  • NEW: conversations.new_per_minute         │
                          └─────────────────────────────────────────────┘
```

> **Key distinction for the new cap:** Layer 2 limits *one* visitor. The new
> `conversations.new_per_minute` is a Layer 3 cap that limits **all visitors combined**
> on a single channel_line — the anti-spam ceiling for an account's inbox.

---

## 2. Two families of limit inside `config/limits.php`

`config/limits.php` mixes two fundamentally different kinds of rule:

| Family | Examples | Enforced by | Stateful? |
|--------|----------|-------------|-----------|
| **Validation** (per-request shape) | `message_chars`, `file_size_kb`, `files_per_request`, `mime_types` | `ChannelLimitsService` | No — pure check of the current request |
| **Rate** (count over time) | `requests_per_minute`, **`conversations.new_per_minute`** | `ChannelRateLimiterService` (Laravel `RateLimiter`) | **Yes — a counter in cache** |

Validation rules ask *"is THIS request allowed?"*. Rate rules ask *"have there been too
many requests in the last minute?"* — that counter is the **pool** this document focuses on.

---

## 3. Anatomy of `config/limits.php`

The file is a tree: **provider → actor → group → key**.

```php
return [
    'website' => [                         // ── provider (channel type)
        'visitor' => [                     // ── actor (visitor | user)
            'messaging' => [               // ── group
                'message_chars'       => 1000,   // validation
                'requests_per_minute' => 20,     // rate  (pool)
            ],
            'uploads' => [
                'file_size_kb'      => 5120,
                'files_per_request' => 3,
                'mime_types'        => [ 'image/jpeg', ... ],
            ],
            'conversations' => [           // ── NEW group
                'new_per_minute' => 30,    // rate  (pool, account-wide)
            ],
        ],
        'user' => [ ... ],
    ],
    'whatsapp' => [ ... ],   'telegram' => [ ... ],
    'viber'    => [ ... ],   'instagram'=> [ ... ],
    'messenger'=> [ ... ],   'email'    => [ ... ],
    'pbx'      => [ 'user' => [ ... ] ],   // no visitor side

    'account' => [                         // ── tenant-level, no provider/actor
        'users'    => ['limit' => 3],
        'widgets'  => ['limit' => 5],
        ...
    ],
];
```

Read with Laravel `data_get()` using dot paths, e.g.
`data_get($limits, 'telegram.visitor.conversations.new_per_minute')`
(see `ChannelRateLimiterService::getLimit()` at `app/Services/Limits/ChannelRateLimiterService.php:62`).

### Providers that have a visitor side (get the new cap)

```
website  whatsapp  telegram  viber  instagram  messenger  email
   └────────────────────── all 7 ──────────────────────┘
pbx → user only, no visitor → no conversation cap
```

---

## 4. How limits are resolved (config vs per-account override)

Limits are **not** read straight from the config file. They pass through
`LimitsResolverService` (`app/Services/Limits/LimitsResolverService.php:15`), which lets a
single account override the global defaults via a `LimitOverride` DB row.

```
                 getEffectiveLimits($accountId)
                            │
            ┌───────────────┴────────────────┐
            ▼                                 ▼
  LimitOverride row for                 no override row
  this account exists?                        │
            │  yes                             │ no
            ▼                                  ▼
   use $override->limits            use config('limits')   ← config/limits.php
            └───────────────┬────────────────┘
                            ▼
              cached in-memory for the request
              (private array $cache[$accountId])
```

**Consequence:** raising the cap for one noisy account = insert/update one `LimitOverride`
row. No deploy, no code change. The new conversation cap inherits this for free.

---

## 5. The pool: how the rate-limit counter fills and resets

This is the heart of your question. The rate family is backed by Laravel's
`Illuminate\Cache\RateLimiter`. It is a **fixed-window counter**, *not* a sliding window
and *not* a gradually-refilling token bucket.

### 5.1 What `hit()` actually does

`RateLimiter::hit($key, 60)` (`vendor/.../Cache/RateLimiter.php:57`) does three things:

```php
$this->cache->add($key.':timer', <now+60s>, 60);  // (1) set window end ONCE (add = if absent)
$this->cache->add($key, 0, 60);                    // (2) create counter = 0 if absent
$this->cache->increment($key);                     // (3) counter += 1
```

- **(1)** uses `add`, so the `:timer` is written **only on the first hit** and is **never
  extended**. The window is anchored to the first request, not the last.
- The counter (`$key`) and the timer both carry a 60-second TTL.

### 5.2 What `tooManyAttempts()` does

```php
if (attempts($key) >= $max) {
    if (cache has $key:timer)  return true;   // window still open → BLOCK
    resetAttempts($key);                      // window expired → wipe counter
}
return false;                                  // → ALLOW (fresh window begins on next hit)
```

### 5.3 The check-then-hit order (in `ChannelRateLimiterService::check`)

`app/Services/Limits/ChannelRateLimiterService.php:35`

```
1. maxPerMinute = configured limit         (≤0 → unlimited, return passed)
2. if tooManyAttempts(key, max)  → REJECT  (the rejected request is NOT counted)
3. else hit(key)                 → consume one slot, ALLOW
```

So the request that *would* be the (N+1)th is rejected **and not added** to the pool.

### 5.4 Timeline visualization (limit = 5 per minute)

Think of the pool as **5 slots that all empty at once** when the 60s window expires —
NOT slots that trickle back one by one.

```
 limit = 5/min            window anchored to FIRST hit ───────────────┐
                                                                       ▼
 time →   t0    t5   t12   t20   t38         (t60: window ends)   t61      t75
 event    req   req  req   req   req   req   req                  req      req
 result   ✓1    ✓2   ✓3    ✓4    ✓5    ✗     ✗                    ✓1       ✓2
 pool    [#----][##---][###--][####-][#####][#####][#####]        [#----]  [##---]
          1/5    2/5    3/5    4/5    5/5   FULL   FULL            reset!   2/5
                                            429    429            new window

          └──────────── one 60s window (t0 → t60) ───────────┘└── next window ──
```

Two important properties this reveals:

1. **No partial refill.** Between t38 and t60 the pool stays FULL — nothing comes back.
   At t60 the timer expires and the *entire* pool empties at once. The first request after
   that (t61) starts a brand-new window.
2. **Window is anchored, not sliding.** If the burst arrived at t0, the reset is at t60 —
   regardless of when the last request came in. A visitor cannot "slide" the window by
   spacing requests out.

### 5.5 `Retry-After`

On rejection, `availableIn($key)` = `timer_expiry − now` (`RateLimiter.php:131`), i.e.
"seconds left in the current window". That value is returned to the caller (HTTP
`Retry-After` header on the widget path).

---

## 6. The NEW cap: conversations per account per channel_line

### 6.1 What it limits

> For a given **account** and a given **channel_line**, at most `new_per_minute` **new**
> conversations may be opened by visitors — **all visitors combined** — in a 60s window.

It does **not** limit messages inside an already-open conversation. It only fires when a
*new* conversation row is about to be created.

### 6.2 The pool key (this is what makes it account-wide)

```
new_conversation_limit:{accountId}:{channelLineId}
                        │           │
                        │           └── separate pool per channel_line (widget A ≠ telegram B)
                        └── the account
        (NO visitor / IP in the key  →  ALL visitors share one pool)
```

Compare to the existing per-actor messaging key, which *does* include the actor:
`channel_limit:{accountId}:{provider}:{actor}:{identifier}` — that pool is per visitor.
Dropping the identifier is the entire trick.

```
 PER-VISITOR pool (Layer 2)                 ACCOUNT-WIDE pool (new Layer 3)
 ┌───────────┐ ┌───────────┐                ┌───────────────────────────────┐
 │ visitor A │ │ visitor B │   ...          │   channel_line 77 (telegram)  │
 │  5/5 full │ │  1/5      │                │   30/30 full ← A+B+C+... summed│
 └───────────┘ └───────────┘                └───────────────────────────────┘
 A is blocked, B still fine                 once full, EVERY visitor blocked
```

### 6.3 Where it is enforced (no single chokepoint)

Conversations are created in 3 places; the cap goes in the two visitor-facing ones:

```
 ┌── website widget ──────────────────────────────────────────────┐
 │ POST /v1/widget/conversations                                   │
 │ ConversationsController@store  (app/.../Widgets/...:48)         │
 │ → check pool BEFORE ->create()   → on FULL: HTTP 429            │
 └────────────────────────────────────────────────────────────────┘

 ┌── social hub (telegram, whatsapp, viber, instagram, messenger, email) ─┐
 │ inbound webhook → queued job → ReceiveMessage                          │
 │ GetConversation::handle()  (app/Actions/SocialHub/GetConversation.php) │
 │ → check pool INSIDE the `if (!$conversation)` (new-only) branch        │
 │ → on FULL: skip create + log via NotifiesLimitsViolation, do NOT throw │
 └────────────────────────────────────────────────────────────────────────┘

 ┌── pbx (phone) ── not a visitor-spam vector → left alone ──┐
 └───────────────────────────────────────────────────────────┘
```

---

## 7. What happens when the pool is full

The behavior differs by path because one is a live HTTP request and the other is a
background job with no client waiting.

| Path | When full | Response | Side effects |
|------|-----------|----------|--------------|
| **Widget** (website) | reject | **HTTP 429** + `Retry-After: <seconds>` JSON body | logged via `NotifiesLimitsViolation` |
| **Social hub** (telegram/email/…) | skip create | webhook already returned 200 to provider; inbound message is **dropped for that window** | logged + Slack alert (throttled) |

### Why "drop, don't throw" on the social-hub side

The webhook job has no user awaiting a response. If we threw an exception:

```
throw ──► queue marks job failed ──► retry/backoff ──► eventually creates the conversation
                                                        anyway → the cap leaks
```

So instead we **skip + log**. The provider already got its 200; the spam message simply
does not open a new conversation this minute.

### Logging / alerting

Both paths funnel through `NotifiesLimitsViolation::notifyAndLog()`
(`app/Http/Middleware/Limits/NotifiesLimitsViolation.php:31`):

```
notifyAndLog()
   ├─ Log::channel('limits')->warning('Limits violation', {account, route, ...})
   └─ Slack webhook  (shouldNotifySlack: 1st hit, then every 10th per account/hour)
```

The Slack throttle (`:64`) prevents a flood: it alerts on the 1st violation and every 10th
within an hour per `{account, message}`.

---

## 8. End-to-end sequence

### Widget (website)

```
Browser            widget route              ChannelRateLimiter         Cache
  │  POST /conversations  │                          │                   │
  │──────────────────────►│  resolve account +       │                   │
  │                       │  channel_line            │                   │
  │                       │  getEffectiveLimits()────────────────────────┤ (override or config)
  │                       │  check(key=acct:line)───►│ tooManyAttempts?──►│
  │                       │                          │◄──── attempts/timer│
  │            ┌──────────┴── passed ──────────┐     │                   │
  │            ▼                                ▼     │                   │
  │   pool NOT full                        pool FULL  │                   │
  │   hit(key) ───────────────────────────────────────────────────────►│ increment
  │   ->create() conversation                 │       │                   │
  │◄── 201 + ConversationResource             │       │                   │
  │                                           ▼       │                   │
  │◄────────────────── 429 + Retry-After ─────┘       │                   │
```

### Social hub (inbound)

```
Provider        webhook ctrl      queue        GetConversation        Cache
  │  POST /social-hub/{acct} │       │                │                 │
  │─────────────────────────►│ 200   │                │                 │
  │◄─────────────────────────┤ dispatch ProcessSocialHubWebhook         │
  │                          │──────►│ ReceiveMessage │                 │
  │                          │       │───────────────►│ existing open    │
  │                          │       │                │ conversation?    │
  │                          │       │         ┌──────┴──────┐           │
  │                          │       │     yes │             │ no        │
  │                          │       │         ▼             ▼           │
  │                          │       │   reuse it    check(acct:line)───►│
  │                          │       │   (no cap)    ┌───────┴───────┐   │
  │                          │       │               ▼               ▼   │
  │                          │       │           NOT full         FULL    │
  │                          │       │           firstOrCreate    skip +  │
  │                          │       │           + hit()          log     │
```

---

## 9. Config reference (proposed values)

`new_per_minute` per provider's `visitor` block. Tune to "max legit new chats/min for a
busy account on that channel"; it is a spam ceiling, set it generously.

| Provider   | suggested `new_per_minute` | rationale |
|------------|---------------------------:|-----------|
| website    | 30 | public widget, highest spam surface |
| whatsapp   | 20 | |
| telegram   | 20 | |
| viber      | 20 | |
| instagram  | 15 | |
| messenger  | 15 | |
| email      | 10 | inbound email is naturally lower-volume |
| pbx        | — | no visitor side |

> `new_per_minute <= 0` (or key absent) ⇒ **unlimited** — same convention as
> `ChannelRateLimiterService::check()` (`:39`).

---

## 10. File map

| Concern | File |
|---------|------|
| Limit values | `config/limits.php` |
| Config-vs-override resolution | `app/Services/Limits/LimitsResolverService.php` |
| Rate counter (the pool) | `app/Services/Limits/ChannelRateLimiterService.php` → Laravel `RateLimiter` |
| Validation (size/mime/chars) | `app/Services/Limits/ChannelLimitsService.php` |
| Account resource caps | `app/Services/Limits/AccountLimitsService.php` |
| Messaging enforcement (existing) | `app/Http/Middleware/Limits/EnforceChannelLimits.php` |
| Logging / Slack alerts | `app/Http/Middleware/Limits/NotifiesLimitsViolation.php` |
| Widget conversation create | `app/Http/Controllers/Widgets/ConversationsController.php:48` |
| Social-hub conversation create | `app/Actions/SocialHub/GetConversation.php:84` |
| **NEW** account-wide cap service | `app/Services/Limits/ConversationCreationLimiterService.php` *(to add)* |
```

> **Status:** Section 6–10 describe the *proposed* conversation cap. The service and the
> two call-site checks are not yet implemented — this doc is the design spec.
