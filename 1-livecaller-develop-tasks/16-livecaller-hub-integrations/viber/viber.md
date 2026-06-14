# Viber Chatbots (REST Bot API)

> Knowledge base entry for integrating Viber as a customer messaging channel. Verified against developers.viber.com and Viber Support docs, June 2026.

## TL;DR

- Viber bots use a single static auth token (`X-Viber-Auth-Token` header) — no OAuth.
- **Webhook only.** There is no polling API; you must register a public HTTPS webhook via `set_webhook` before you can do anything.
- Inbound callbacks are signed with `X-Viber-Content-Signature` (HMAC-SHA256 of the raw body, keyed with your auth token).
- You can only message **subscribed** users freely (plus one welcome message after `conversation_started`). A user becomes subscribed by sending the bot their first message.
- **Commercial model:** since **February 5, 2024**, Viber bots can only be created on commercial terms — apply through Viber/Rakuten or a verified partner, pay a monthly maintenance fee (EUR 100/bot as of the published terms), and pay per *chatbot-initiated* message. Messages inside an active 24-hour session are free. Verify current pricing with Viber before committing — rates are per recipient country and the model has changed before (there is a "legacy" model for older bots).
- User IDs are **scoped per bot** — the same person has a different ID in each bot.

## How the integration works (setup steps)

1. **Get a bot.** As of 05.02.2024, you can no longer freely self-serve a bot. Apply to Rakuten Viber directly or via a verified partner (start at https://www.forbusiness.viber.com/en/chatbots/ ; historically partners.viber.com was the self-serve portal). You agree to the commercial terms (monthly maintenance fee + per-message rates for chatbot-initiated messages).
2. **Get the auth token.** The token is generated when the bot is created. It is also visible in the Viber app under *More > Settings > Bots > Edit Info > Your app key*. Store it as a secret; it is the only credential.
3. **Expose a public HTTPS endpoint** with a valid (not self-signed) SSL certificate to receive callbacks.
4. **Register the webhook** with `POST https://chatapi.viber.com/pa/set_webhook`. Viber immediately sends a callback with `"event": "webhook"` to your URL — you must respond `200 OK` or registration fails.
5. **Handle inbound events** (`message`, `subscribed`, `unsubscribed`, `conversation_started`, `delivered`, `seen`, `failed`) and validate the signature on every request.
6. **Send messages** with `POST https://chatapi.viber.com/pa/send_message` to subscribed users, mapping Viber user IDs to your conversation/contact records.

## Auth / token model

- Every API call requires the header `X-Viber-Auth-Token: <token>`. Missing header → `missing_auth_token` error; bad token → status code `2` (invalid auth token).
- One static token per bot. No OAuth, no refresh, no scopes. Rotation = regenerating via Viber.
- All endpoints live under `https://chatapi.viber.com/pa/` and use `POST` with JSON bodies. Every response contains `status` (0 = success) and `status_message`.

## Receiving messages (webhook)

Webhooks are **mandatory** — Viber has no `getUpdates`-style polling.

### Setting the webhook

```http
POST https://chatapi.viber.com/pa/set_webhook
X-Viber-Auth-Token: 445da6az1s345z78-dazcczb2542zv51a-e0vc5fve17480im9
Content-Type: application/json

{
  "url": "https://example.com/webhooks/viber",
  "event_types": ["delivered", "seen", "failed", "subscribed", "unsubscribed", "conversation_started"],
  "send_name": true,
  "send_photo": true
}
```

- `event_types` is optional filtering; `message`, `subscribed` and `unsubscribed` are **always delivered** and cannot be opted out of.
- To remove the webhook, call `set_webhook` with an empty `"url": ""`.
- Failed deliveries are retried up to 10 times with backoff (10s, 60s, 300s, 600s, then 900s intervals). Respond `200` fast; do heavy work async.

### Webhook events

| Event | Meaning |
|---|---|
| `webhook` | Sent once when you register the URL (must answer 200). |
| `conversation_started` | User opened a 1:1 chat with the bot (via "Message" button or deep link). **Not a subscription.** You may reply with one welcome message (within ~5 minutes), free of charge. |
| `subscribed` | User subscribed (opened 1:1 conversation, or pressed "Receive messages"). Note: a user who subscribes by *sending a first message* triggers `message`, not `subscribed`. |
| `unsubscribed` | User unsubscribed — stop messaging them (`send_message` will return status 6). |
| `message` | Inbound user message. Types: `text`, `picture`, `video`, `file`, `sticker`, `contact`, `url`, `location`. Carries back your `tracking_data` if set. |
| `delivered` / `seen` / `failed` | Delivery receipts per `message_token`. |

Example inbound message callback:

```json
{
  "event": "message",
  "timestamp": 1457764197627,
  "message_token": 4912661846655238145,
  "sender": {
    "id": "01234567890A=",
    "name": "John McClane",
    "avatar": "https://avatar.example.com",
    "country": "UK",
    "language": "en",
    "api_version": 1
  },
  "message": {
    "type": "text",
    "text": "a message to the service",
    "tracking_data": "conversation-id-123"
  }
}
```

### Signature validation

Every callback carries `X-Viber-Content-Signature`: hex-encoded **HMAC-SHA256 of the raw request body, keyed with your auth token**. Always validate before processing (PHP example, since our stack is Laravel):

```php
$signature = $request->header('X-Viber-Content-Signature');
$expected  = hash_hmac('sha256', $request->getContent(), config('services.viber.token'));

if (! hash_equals($expected, (string) $signature)) {
    abort(401, 'Invalid Viber signature');
}
```

(Some Viber transports also append `?sig=` as a query param — the header is the documented mechanism.)

## Sending messages

```http
POST https://chatapi.viber.com/pa/send_message
X-Viber-Auth-Token: <token>
Content-Type: application/json

{
  "receiver": "01234567890A=",
  "min_api_version": 1,
  "sender": { "name": "Acme Support", "avatar": "https://acme.com/avatar.jpg" },
  "tracking_data": "conversation-id-123",
  "type": "text",
  "text": "Hello from support!"
}
```

Success response:

```json
{ "status": 0, "status_message": "ok", "message_token": 5741311803571721087, "billing_status": 1 }
```

- `receiver` must be a **subscribed** user's ID; otherwise status `6` ("Not subscribed"). Status `5` = not a Viber user.
- Max request size 30KB. `sender.name` ≤ 28 chars; `sender.avatar` ≤ 100KB (720×720 recommended).
- `tracking_data` (≤ 4096 chars) is echoed back on the user's next reply — useful to correlate replies to a helpdesk conversation.
- `billing_status` (0–5) tells you whether the message was billed as session vs chatbot-initiated under the commercial model.

### Message types

`text`, `picture` (jpeg/png, with `media` + optional `thumbnail`), `video` (`media`, `size`, `duration`, `thumbnail`), `file` (`media`, `size`, `file_name`; executables like .exe/.apk/.bat are blocked), `contact`, `location` (`lat`/`lon`), `url`, `sticker` (`sticker_id`), and `rich_media` (carousel — button grid up to 6 columns × 7 rows per item; button `ActionType`: `open-url`, `reply`; `location-picker`/`share-phone` are keyboard-only). Any message can attach a `keyboard` object (custom reply keyboard shown to the user). Richer types may require a higher `min_api_version` — devices below it fail with status `13`.

### Broadcast

`POST https://chatapi.viber.com/pa/broadcast_message` — same message schema, with `broadcast_list` of up to **300 subscribed user IDs** per call; rate-limited to **500 requests per 10 seconds**. Response includes a `failed_list`. Supports per-recipient placeholders (`replace_me_with_receiver_id`, `replace_me_with_url_encoded_receiver_id`, `replace_me_with_user_name`).

### Other useful endpoints

- `POST /pa/get_account_info` — bot details + subscriber count.
- `POST /pa/get_user_details` — profile by user ID (**heavily limited: 2 requests per user per 12 hours**).
- `POST /pa/get_online` — online status for up to 100 user IDs.

## Messaging rules, sessions and billing

This is **not** the WhatsApp "24-hour window or nothing" model, but since 2024 it's economically similar:

- **Subscribed users** can always be messaged via the API (no technical time window). But under the commercial model, *what you pay* depends on session state:
  - **Session** = a 24-hour window opened when the **user** messages the bot (or replies to a chatbot-initiated message). Messages within a session are **free**, with a cap of **60 messages per session**; after 60 the session ends.
  - **Chatbot-initiated message** = any bot→subscriber message outside an active session. **Billed** per message, rate based on the country code of the subscriber's phone number.
  - **Welcome message** (the single reply allowed after `conversation_started`) is **free** and is the only thing you may send to a not-yet-subscribed user.
- **Not-subscribed users**: cannot be messaged at all, except the one welcome message within ~5 minutes of `conversation_started`.
- Monthly **maintenance fee** of EUR 100 per bot (per published terms; may differ by contract/region — confirm with Viber, older bots may be on the "legacy" model).
- Practical rate limits: broadcast 500 req/10s; general send_message limits exist (status `12` = rate limit breach) but Viber does not publish a precise global msg/sec figure — back off on status 12.

## Limitations / gotchas

- No polling — local dev needs a tunnel (ngrok etc.) and webhook re-registration.
- User IDs are per-bot; you can't correlate a user across two bots or get their phone number.
- No way to fetch message history; if you miss a webhook (after retries are exhausted), the message is gone.
- A `conversation_started` user is **not** subscribed (`"subscribed": false` in the payload) — don't create a contact as reachable until they actually subscribe/message.
- Media is sent by **URL** (Viber fetches it) — you must host attachments on a public HTTPS URL; inbound media likewise arrives as a Viber-hosted URL with limited lifetime.
- Onboarding friction: bot creation is gated behind a commercial application since Feb 2024 — factor lead time and cost into customer onboarding.
- **Viber Channels ≠ bots**: Channels are one-to-many broadcast feeds (like Telegram channels) with their own Channels Post API; two-way support conversations require a **bot**.

## Key Terms

- **Auth token** — static secret identifying the bot; sent as `X-Viber-Auth-Token` on every API call and used as the HMAC key for webhook signatures.
- **Subscriber** — a user who messaged the bot or pressed "Receive messages"; the only kind of user you can message via `send_message`.
- **`conversation_started`** — event when a user opens the chat; allows exactly one free welcome message; not a subscription.
- **Session** — 24-hour, max-60-message window opened by a user message; messages inside it are free under the commercial model.
- **Chatbot-initiated message** — bot→user message outside a session; billed per recipient country.
- **`tracking_data`** — opaque string (≤4096 chars) attached to an outbound message and echoed back on the user's reply.
- **`min_api_version`** — minimum Viber client API level required to render the message; older clients fail with status 13.
- **`message_token`** — Viber's ID for a message, referenced by `delivered`/`seen`/`failed` receipts.
- **Viber Channel** — one-way broadcast feed; distinct product from bots.

## Official Docs

- REST Bot API reference: https://developers.viber.com/docs/api/rest-bot-api/
- Developers hub: https://developers.viber.com/
- Bots for business / signup: https://www.forbusiness.viber.com/en/chatbots/
- Bot commercial model: https://help.viber.com/hc/en-us/articles/15247629658525-Bot-commercial-model
- Commercial model FAQ: https://help.viber.com/hc/en-us/articles/15383950711197-Rakuten-Viber-chatbot-commercial-model-FAQ
- Legacy commercial model (pre-2024 bots): https://help.viber.com/hc/en-us/articles/8746671603485-Chatbot-commercial-model-legacy
