# Customer-Facing Outbound Webhooks — Architecture & Implementation Plan

> **Goal:** Let customers register **one or more** webhook endpoints, each with its **own secret** and its **own subscription to a set of event types**. When a subscribed event fires, LiveCaller signs and delivers a JSON payload to every matching endpoint, with retries, delivery logging, and an admin/customer UI to manage them.

This doc is written to fit the **existing LiveCaller API conventions** (Laravel 7 / PHP 7.4, `account_id` multi-tenancy, Horizon/Redis queues, Passport OAuth, Spatie packages, Nova admin).

---

## 1. Current State (what exists today)

LiveCaller already does **inbound** webhooks but has **no first-class outbound webhook system**.

| Area | Status | Files |
|------|--------|-------|
| Inbound webhooks | ✅ Uses `spatie/laravel-webhook-client` | `config/webhook-client.php`, `app/Http/Controllers/Webhooks/*`, `app/Jobs/ProcessSocialHubWebhook.php`, `ProcessPBXWebhook.php` |
| Outbound (current) | ⚠️ One ad-hoc job, single hard-coded event, **MD5** token (not HMAC), single URL stored in account `settings` | `app/Jobs/ProcessConversationWebHook.php` |
| Outbound (target) | ❌ Multi-endpoint, multi-event, per-endpoint HMAC secret, delivery log, UI | *to build* |

**`ProcessConversationWebHook` (the thing we're replacing/generalizing):**
```php
// Fires only on conversation_closed; one URL from account setting; md5 token
$settings = $this->conversation->getCacheAccount()->setting('webhooks.conversation_closed');
$payload = [
    'token' => md5($conversationId . $widget->extra_attributes->get('webhook_secret')),
    'conversation_id' => ...,
];
Http::withHeaders(['User-Agent' => null])->retry(2, 3000)->post($url, $payload);
```

The new system **subsumes** this: `conversation.closed` becomes one event type among many, delivered through the generic pipeline.

> **Decision needed:** add `spatie/laravel-webhook-server` (battle-tested signer/dispatcher/retry-backoff) **or** hand-roll a small dispatcher. Recommendation below assumes a **thin custom layer** so we control the DB schema, per-endpoint secrets, and delivery logging — but we can wrap `webhook-server` for the actual HTTP call + signing if preferred. Either way the model/DB design is identical.

---

## 2. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Domain action happens (conversation closed, message created, tag …)   │
│  → Laravel Event fired (existing app/Events/*)                          │
└───────────────────────────────┬────────────────────────────────────────┘
                                 │
                  ┌──────────────▼───────────────┐
                  │  WebhookEventListener          │  (subscribes to mapped events)
                  │  - maps Event class → event    │
                  │    key (e.g. "conversation     │
                  │    .closed")                   │
                  │  - builds canonical payload    │
                  │    via a WebhookResource        │
                  └──────────────┬─────────────────┘
                                 │  account_id + event key + payload
                  ┌──────────────▼───────────────┐
                  │  WebhookDispatcher (service)   │
                  │  - find enabled endpoints for  │
                  │    account subscribed to key   │
                  │  - create webhook_deliveries   │
                  │    rows (status=pending)        │
                  │  - queue one job per delivery   │
                  └──────────────┬─────────────────┘
                                 │
                  ┌──────────────▼───────────────┐
                  │  SendWebhookJob (queue:webhooks)│  ShouldQueue
                  │  - load endpoint + delivery     │
                  │  - sign body (HMAC-SHA256)      │
                  │  - POST with timeout            │
                  │  - record response/attempt      │
                  │  - retry w/ backoff or mark     │
                  │    failed; auto-disable endpoint│
                  │    after N consecutive fails    │
                  └─────────────────────────────────┘
```

### Why an event listener (not calling the dispatcher everywhere)
The codebase already fires rich domain events (`ConversationClosed`, `ConversationCreated`, `MemberAssignedToConversation`, …). We hook the webhook system onto those events via **one subscriber**, so we don't sprinkle webhook calls across controllers/services. New event types are added by registering a mapping, not by editing business logic.

### Tenancy
Everything is scoped by **`account_id`** (the primary tenant key — `bigInteger unsigned`, FK to `accounts`, `onDelete CASCADE`), matching `users`, `widgets`, `conversations`.

### Queue
Use a dedicated **`webhooks`** queue on the existing Redis/Horizon setup (add to a supervisor's queue list in `config/horizon.php`). Keeps outbound HTTP off the `default`/`high` lanes. Falls back fine to `low` if we don't want a new supervisor.

---

## 3. Database Structure

Three tables. Matches existing conventions: `bigIncrements` PK, `account_id` FK with cascade, `timestamps`, `softDeletes` where useful, JSON columns for flexible config.

### 3.1 `webhook_endpoints` — the customer-registered listener

```php
// database/migrations/2026_06_15_000001_create_webhook_endpoints_table.php
Schema::create('webhook_endpoints', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->bigInteger('account_id')->unsigned();

    $table->string('name')->nullable();                 // friendly label, e.g. "Zapier prod"
    $table->string('url', 2048);                         // delivery target (https enforced in validation)
    $table->string('secret', 64);                        // per-endpoint signing secret (generated)

    $table->boolean('is_active')->default(true);         // customer on/off toggle
    $table->boolean('is_disabled_by_system')->default(false); // auto-disabled after repeated failures
    $table->string('api_version')->default('v1');        // payload schema version for this endpoint

    // event subscriptions: array of event keys, e.g. ["conversation.closed","conversation.created"]
    // "*" means "all events"
    $table->json('subscribed_events');

    $table->unsignedInteger('consecutive_failures')->default(0);
    $table->timestamp('last_success_at')->nullable();
    $table->timestamp('last_failure_at')->nullable();
    $table->string('last_failure_reason')->nullable();

    $table->json('settings')->nullable();                // headers, custom labels, future flags
    $table->extraAttributes();                           // schemaless macro used across the codebase

    $table->timestamps();
    $table->softDeletes();

    $table->foreign('account_id')->references('id')->on('accounts')
        ->onUpdate('CASCADE')->onDelete('CASCADE');

    $table->index(['account_id', 'is_active']);
});
```

**Notes**
- `secret` is generated server-side (`Str::random(40)`), shown to the customer once (and re-fetchable in UI). Used for HMAC. Can be **rotated** — keep `secret` + optional `secret_previous` if we want zero-downtime rotation (add later).
- `subscribed_events` as JSON keeps one row per endpoint. If you prefer relational filtering/reporting per event, use the pivot in 3.2 instead (pick **one** approach — JSON is simpler and matches the codebase's heavy `settings`/`extra_attributes` JSON usage).

### 3.2 *(Optional)* `webhook_endpoint_events` — relational subscription pivot

Use **instead of** the `subscribed_events` JSON column if you want SQL-level "who listens to event X" queries and per-event stats.

```php
Schema::create('webhook_endpoint_events', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->unsignedBigInteger('webhook_endpoint_id');
    $table->string('event_key');                         // e.g. "conversation.closed"
    $table->timestamps();

    $table->foreign('webhook_endpoint_id')->references('id')->on('webhook_endpoints')
        ->onUpdate('CASCADE')->onDelete('CASCADE');
    $table->unique(['webhook_endpoint_id', 'event_key']);
    $table->index('event_key');
});
```

> **Recommendation:** start with the **JSON column** (3.1). Promote to the pivot only if reporting needs it.

### 3.3 `webhook_deliveries` — per-attempt delivery log

```php
Schema::create('webhook_deliveries', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->bigInteger('account_id')->unsigned();
    $table->unsignedBigInteger('webhook_endpoint_id');

    $table->uuid('event_id');                            // idempotency key, sent in payload + header
    $table->string('event_key');                         // "conversation.closed"

    $table->json('payload');                             // exact body sent (or a trimmed copy)
    $table->string('status')->default('pending');        // pending|success|failed
    $table->unsignedTinyInteger('attempts')->default(0);

    $table->unsignedSmallInteger('response_status')->nullable(); // HTTP code from target
    $table->text('response_body')->nullable();           // truncated (e.g. first 2KB)
    $table->unsignedInteger('duration_ms')->nullable();

    $table->timestamp('delivered_at')->nullable();
    $table->timestamp('next_retry_at')->nullable();

    $table->timestamps();

    $table->foreign('account_id')->references('id')->on('accounts')
        ->onUpdate('CASCADE')->onDelete('CASCADE');
    $table->foreign('webhook_endpoint_id')->references('id')->on('webhook_endpoints')
        ->onUpdate('CASCADE')->onDelete('CASCADE');

    $table->index(['webhook_endpoint_id', 'status']);
    $table->index(['account_id', 'event_key']);
    $table->index('event_id');
});
```

**Retention:** deliveries grow fast. Add a scheduled prune (e.g. delete `success` older than 30 days, `failed` older than 90 days) via `app/Console` + Kernel schedule. Consider partitioning/archival only if volume demands it.

---

## 4. Event Types (the catalog customers subscribe to)

Event keys are stable, dotted, namespaced strings — **decoupled** from internal class names so we can refactor freely. Define them in an enum + a registry that maps a Laravel event class → key + payload builder.

### 4.1 Proposed `WebhookEventType` enum (`app/Enums/`)

```php
final class WebhookEventType
{
    // Conversations
    const CONVERSATION_CREATED       = 'conversation.created';
    const CONVERSATION_CLOSED        = 'conversation.closed';
    const CONVERSATION_REOPENED      = 'conversation.reopened';
    const CONVERSATION_TRANSFERRED   = 'conversation.transferred';
    const CONVERSATION_HELD          = 'conversation.held';
    const CONVERSATION_UNHELD        = 'conversation.unheld';

    // Messages
    const MESSAGE_CREATED            = 'message.created';
    const EMAIL_MESSAGE_COMPLETED    = 'message.email_completed';

    // Membership / assignment
    const MEMBER_ASSIGNED            = 'conversation.member_assigned';
    const USER_ATTACHED              = 'conversation.user_attached';
    const PARTICIPANT_DETACHED       = 'conversation.participant_detached';

    // Tags
    const TAG_ATTACHED               = 'conversation.tag_attached';
    const TAG_DETACHED               = 'conversation.tag_detached';

    // Visitors / contacts
    const VISITOR_UPDATED            = 'visitor.updated';

    // Callbacks / PBX (telephony)
    const CALLBACK_CREATED           = 'callback.created';
    const CALL_STARTED               = 'pbx.call_started';
    const CALL_ENDED                 = 'pbx.call_ended';

    // Knowledge base
    const KB_ARTICLE_CREATED         = 'kb.article_created';

    public static function all(): array { /* return all constants */ }
}
```

### 4.2 Mapping table — internal event → webhook key

| Internal Laravel event | Webhook event key | Notes |
|---|---|---|
| `App\Events\Conversation\ConversationCreated` | `conversation.created` | |
| `App\Events\Conversation\ConversationClosed` | `conversation.closed` | replaces `ProcessConversationWebHook` |
| `App\Events\Conversation\ConversationReopened` | `conversation.reopened` | |
| `App\Events\Conversation\ConversationTransferred` (+ `...ToDepartment`) | `conversation.transferred` | include `to` in payload |
| `App\Events\Conversation\ConversationHeld` / `...Unheld` | `conversation.held` / `.unheld` | |
| `App\Events\Conversation\ConversationMessageCreated` | `message.created` | high volume — see throttling note |
| `App\Events\Conversation\ConversationEmailMessageCompleted` | `message.email_completed` | |
| `App\Events\MemberAssignedToConversation` | `conversation.member_assigned` | member can be User/AI |
| `App\Events\UserAttachedToConversation` | `conversation.user_attached` | |
| `App\Events\TagAttached` / `TagDetached` | `conversation.tag_attached` / `.tag_detached` | |
| `App\Events\VisitorUpdated` | `visitor.updated` | |
| `App\Events\CallbackCreated` | `callback.created` | |
| `App\Events\PBX\*` | `pbx.call_started` / `pbx.call_ended` | confirm exact class names in `app/Events/PBX/` |
| `App\Events\KbCreated` | `kb.article_created` | |

> **Volume warning:** `message.created` can be extremely chatty (every chat message). Make it opt-in, and consider per-account rate limiting / batching for it specifically.

### 4.3 Payload contract (versioned, `v1`)

Every delivery body is the same envelope:

```json
{
  "id": "8f3c2b9e-1c2a-4d5e-9f00-aabbccddeeff",
  "event": "conversation.closed",
  "api_version": "v1",
  "created_at": "2026-06-15T12:34:56Z",
  "account_id": 4821,
  "data": {
    "conversation": {
      "id": 12345,
      "type": "chat",
      "site_name": "Acme Support",
      "tags": ["billing", "vip"],
      "operators": [{ "id": 9, "email": "agent@acme.com", "name": "Sam" }],
      "visitor": { "name": "Jane", "email": "jane@x.com", "mobile": null },
      "closed_by": { "type": "user", "id": 9 }
    }
  }
}
```

- `data` shape per event is built by **dedicated webhook resources** (mirror the existing `JsonResource` transformer pattern, e.g. `ConversationResource`). **Do not** reuse the websocket `broadcastWith()` payloads verbatim — those are tuned for the dashboard and may leak internal fields. Define lean, documented `app/Http/Resources/Webhooks/*` resources.
- `id` is the idempotency key (= `webhook_deliveries.event_id`). Customers should dedupe on it.
- `api_version` lets us evolve payloads without breaking existing endpoints.

---

## 5. Security — Signing & Verification

Per-endpoint **HMAC-SHA256** over the **raw request body** (upgrade from the current MD5 token).

**Headers sent with every delivery:**
```
Content-Type: application/json
User-Agent: LiveCaller-Webhooks/1.0
X-LiveCaller-Event: conversation.closed
X-LiveCaller-Delivery: 8f3c2b9e-...        # event_id / idempotency key
X-LiveCaller-Signature: t=1718456096,v1=<hex hmac>
```

**Signature scheme (Stripe-style, replay-resistant):**
```
signed_payload = "{timestamp}.{raw_body}"
v1 = hex( hmac_sha256(signed_payload, endpoint.secret) )
header = "t={timestamp},v1={v1}"
```

**Customer verification (example, what we publish in customer docs):**
```php
$timestamp = /* parse t= from header */;
$expected  = hash_hmac('sha256', $timestamp . '.' . $rawBody, $endpointSecret);
$valid = hash_equals($expected, $v1) && abs(time() - $timestamp) < 300; // 5-min tolerance
```

- **HTTPS only** — reject non-https URLs at validation time.
- **SSRF protection** — when a customer saves a URL, resolve & block private/loopback/link-local ranges (no `localhost`, `169.254.*`, `10.*`, `192.168.*`, metadata `169.254.169.254`). Re-check at send time.
- **Secret rotation** — allow regenerating the secret; optionally accept old secret for a grace window.
- **No secrets in logs** — never store the secret in `webhook_deliveries`; sign at send time.

---

## 6. Delivery, Retries & Failure Handling

`SendWebhookJob` (`ShouldQueue`, queue `webhooks`):

- **Timeout:** ~10s connect+read (`Http::timeout(10)`).
- **Retries:** exponential backoff. With `webhook-server` you get this built-in; hand-rolled, use `public $backoff = [60, 300, 1800, 7200, 21600];` (1m,5m,30m,2h,6h) and `public $tries = 6`.
- **Success:** any `2xx`. Record `response_status`, `duration_ms`, set delivery `success`, reset `consecutive_failures`, set `last_success_at`.
- **Failure:** non-2xx / timeout / connection error → increment attempt, schedule retry, set `next_retry_at`.
- **Auto-disable:** after N consecutive failures (e.g. 15) across deliveries, set `is_disabled_by_system = true`, stop sending, and notify the customer (email/notification). Customer re-enables manually after fixing their endpoint.
- **Idempotency:** include stable `event_id`; document that customers must dedupe (we retry, so duplicates are possible).
- **Ordering:** not guaranteed (parallel queue). Customers should rely on `created_at`/their own state, not arrival order.

---

## 7. Code Components to Build

```
app/
├── Enums/WebhookEventType.php                      # event key constants + all()
├── Models/WebhookEndpoint.php                      # belongsTo Account; casts subscribed_events:array
├── Models/WebhookDelivery.php                      # belongsTo Endpoint, Account
├── Services/Webhooks/WebhookDispatcher.php         # find endpoints + create deliveries + queue
├── Services/Webhooks/WebhookEventRegistry.php      # map event class → key + resource builder
├── Listeners/DispatchWebhooksListener.php          # subscriber on the mapped domain events
├── Jobs/SendWebhookJob.php                         # signed HTTP POST + retry/record
├── Http/
│   ├── Controllers/WebhookEndpointsController.php  # customer CRUD (Passport-scoped)
│   ├── Requests/Webhooks/StoreWebhookEndpointRequest.php
│   ├── Requests/Webhooks/UpdateWebhookEndpointRequest.php
│   └── Resources/
│       ├── WebhookEndpointResource.php
│       └── Webhooks/ConversationWebhookResource.php  # lean per-event payloads
├── Nova/WebhookEndpoint.php                         # admin management (mirror Account.php style)
database/migrations/
│   ├── *_create_webhook_endpoints_table.php
│   └── *_create_webhook_deliveries_table.php
config/webhook.php                                   # event catalog, queue name, retry policy, limits
```

### 7.1 Registry + Listener sketch

```php
// WebhookEventRegistry: class => [key, resourceBuilder, accountResolver]
$registry = [
    ConversationClosed::class => [
        'key'      => WebhookEventType::CONVERSATION_CLOSED,
        'account'  => fn($e) => $e->conversation->account_id,
        'data'     => fn($e) => (new ConversationWebhookResource($e->conversation))->resolve(),
    ],
    // ...
];

// DispatchWebhooksListener::handle($event)
$map = $registry[get_class($event)] ?? null;
if (!$map) return;
app(WebhookDispatcher::class)->dispatch(
    accountId: ($map['account'])($event),
    eventKey:  $map['key'],
    data:      ($map['data'])($event),
);
```

```php
// WebhookDispatcher::dispatch()
$endpoints = WebhookEndpoint::query()
    ->where('account_id', $accountId)
    ->where('is_active', true)
    ->where('is_disabled_by_system', false)
    ->get()
    ->filter(fn ($e) => in_array('*', $e->subscribed_events, true)
        || in_array($eventKey, $e->subscribed_events, true));

foreach ($endpoints as $endpoint) {
    $delivery = WebhookDelivery::create([
        'account_id'          => $accountId,
        'webhook_endpoint_id' => $endpoint->id,
        'event_id'            => (string) Str::uuid(),
        'event_key'           => $eventKey,
        'payload'             => $envelope,   // built once
        'status'              => 'pending',
    ]);
    SendWebhookJob::dispatch($delivery->id)->onQueue('webhooks');
}
```

> Register the listener as an **event subscriber** in `EventServiceProvider::$subscribe` so one class maps many events. Because the domain events already exist, **no business logic changes are required** to start emitting webhooks.

---

## 8. Customer-Facing API (Passport, `v1`)

Mirror the existing scope-based controller pattern (`scope:resource:action`, `auth()->user()->account_id`).

```php
// routes/api.php — inside the v1 group
Route::prefix('webhook-endpoints')->name('webhook-endpoints.')->group(function () {
    Route::get('/',            [WebhookEndpointsController::class, 'index'])->middleware('scope:webhooks:read');
    Route::post('/',           [WebhookEndpointsController::class, 'store'])->middleware('scope:webhooks:create');
    Route::get('{endpoint}',   [WebhookEndpointsController::class, 'show'])->middleware('scope:webhooks:read');
    Route::put('{endpoint}',   [WebhookEndpointsController::class, 'update'])->middleware('scope:webhooks:update');
    Route::delete('{endpoint}',[WebhookEndpointsController::class, 'destroy'])->middleware('scope:webhooks:delete');

    Route::post('{endpoint}/rotate-secret', [WebhookEndpointsController::class, 'rotateSecret'])->middleware('scope:webhooks:update');
    Route::post('{endpoint}/test',          [WebhookEndpointsController::class, 'sendTest'])->middleware('scope:webhooks:update');
    Route::get('{endpoint}/deliveries',     [WebhookEndpointsController::class, 'deliveries'])->middleware('scope:webhooks:read');
});
Route::get('webhook-events', [WebhookEndpointsController::class, 'availableEvents'])->middleware('scope:webhooks:read');
```

**Validation (`StoreWebhookEndpointRequest`)** — follow the existing FormRequest style:
```php
public function rules(): array {
    return [
        'name'               => ['nullable', 'string', 'max:255'],
        'url'                => ['required', 'url', 'max:2048', 'starts_with:https://', new PublicUrlRule()],
        'subscribed_events'  => ['required', 'array', 'min:1'],
        'subscribed_events.*'=> ['required', 'string', Rule::in(array_merge(['*'], WebhookEventType::all()))],
        'is_active'          => ['boolean'],
    ];
}
```
- `authorize()` + a `WebhookEndpointPolicy` (or `BelongsToAccount` rule) so customers only touch their own account's endpoints.
- `secret` is returned **only** on create / rotate-secret (write-only thereafter, like a password).
- Enforce a per-account **max endpoints** limit (reuse the `account.limit:*` middleware pattern).
- `sendTest` delivers a `webhook.test` event so customers can validate their receiver.

---

## 9. Admin (Nova)

`app/Nova/WebhookEndpoint.php` mirroring `app/Nova/Account.php`:

```php
public function fields(Request $request) {
    return [
        ID::make()->sortable(),
        BelongsTo::make('Account')->searchable(),
        Text::make('Name'),
        Text::make('URL')->hideFromIndex(),
        Boolean::make('Active', 'is_active'),
        Boolean::make('Disabled by system', 'is_disabled_by_system'),
        Code::make('Subscribed events', 'subscribed_events')->json(),
        Number::make('Consecutive failures'),
        DateTime::make('Last success at')->exceptOnForms(),
        DateTime::make('Last failure at')->exceptOnForms(),
        Text::make('Last failure reason')->hideFromIndex(),
        HasMany::make('Deliveries', 'deliveries', WebhookDelivery::class),
    ];
}
```
- Add a Nova action **"Re-enable endpoint"** (clears `is_disabled_by_system`, resets `consecutive_failures`).
- Optional dashboard card: failed deliveries in last 24h.

---

## 10. Config (`config/webhook.php`)

```php
return [
    'queue'      => env('WEBHOOK_QUEUE', 'webhooks'),
    'timeout'    => 10,
    'tries'      => 6,
    'backoff'    => [60, 300, 1800, 7200, 21600],
    'max_endpoints_per_account' => 10,
    'auto_disable_after_failures' => 15,
    'signature_tolerance_seconds' => 300,
    'response_body_max_bytes' => 2048,
    'prune' => ['success_days' => 30, 'failed_days' => 90],
];
```
Add `webhooks` to a Horizon supervisor's `queue` array in `config/horizon.php`.

---

## 11. Migration / Rollout Plan

1. **Schema + models + config** (no behavior change yet).
2. **Dispatcher + listener + job**, wired to **one low-risk event** (`conversation.closed`) behind a feature flag / account setting.
3. **Backfill parity:** make new pipeline reproduce `ProcessConversationWebHook` output for `conversation.closed`, then **deprecate & remove** the old job and its `settings('webhooks.conversation_closed')` usage.
4. **Expand event catalog** (add registry entries + payload resources incrementally).
5. **Customer API + Nova UI.**
6. **Delivery log pruning** scheduled command.
7. **Docs** for customers (signing verification snippet, event catalog, payload examples, retry/idempotency behavior).

---

## 12. Open Questions / Decisions

- [ ] Use `spatie/laravel-webhook-server` for sign+dispatch+retry, or hand-roll? (DB/model design is the same either way.)
- [ ] `subscribed_events` as **JSON column** (simple, recommended) vs **pivot table** (better reporting)?
- [ ] Is the listener keyed on **account** or should some events be scoped to a **widget** (and thus an endpoint subscribable per-widget)? Current design is account-wide; per-widget filtering can be a `settings` filter on the endpoint later.
- [ ] Rate limiting / batching policy for high-volume `message.created`.
- [ ] Retention window + whether to expose delivery logs to customers (recommended: yes, read-only, last 7–30 days).

---

### TL;DR
- **3 tables:** `webhook_endpoints` (account-scoped, per-endpoint secret + `subscribed_events` JSON), `webhook_deliveries` (per-attempt log), optional `webhook_endpoint_events` pivot.
- **Pipeline:** existing domain Events → `DispatchWebhooksListener` → `WebhookDispatcher` (fan-out to subscribed endpoints) → `SendWebhookJob` on a `webhooks` queue → HMAC-SHA256 signed POST with retries/backoff + delivery logging + auto-disable.
- **Security:** per-endpoint HMAC (replaces MD5), HTTPS-only, SSRF guard, replay-tolerant signature, idempotency key.
- **Surfaces:** Passport-scoped `v1/webhook-endpoints` CRUD + test/rotate/deliveries, plus a Nova admin resource.
- **Generalizes** the existing single-purpose `ProcessConversationWebHook` into a multi-endpoint, multi-event system.
