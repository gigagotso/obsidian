# Telegram Bot API

> Knowledge base entry for integrating Telegram as a customer messaging channel. Verified against core.telegram.org, June 2026 (current Bot API version: 10.0, released 2026-05-08).

## TL;DR

- Bots are created in ~30 seconds via **@BotFather** (`/newbot`); the only credential is the **bot token** (`123456:ABC-DEF1234ghIkl-...`). No OAuth, no app review.
- Two ways to receive updates, **mutually exclusive**: long polling (`getUpdates`) or HTTPS webhook (`setWebhook`). Webhook validation uses the `X-Telegram-Bot-Api-Secret-Token` header (no HMAC signature).
- Send via `POST https://api.telegram.org/bot<token>/sendMessage` (and other `send*` methods) using `chat_id`.
- **Bots cannot message a user first** — the user must press Start (send `/start`) before the bot can ever reach them. After that there is no time window: the bot can message them anytime, forever, unless blocked.
- Rate limits: ~1 msg/sec per chat, 20 msg/min per group, ~30 msg/sec global (up to 1000/sec with paid broadcasts at 0.1 Stars/msg).
- File limits via the hosted Bot API: download ≤ 20 MB (`getFile`), upload ≤ 50 MB. Run your own Bot API server to lift these (uploads up to 2000 MB).

## Two APIs — don't confuse them

1. **Bot API** (`https://api.telegram.org/bot<token>/...`) — simple HTTPS/JSON API for bots. Auth = bot token. This is what a helpdesk integration uses.
2. **MTProto client API** (via **TDLib**) — the full client protocol used by Telegram apps. Auth = `api_id`/`api_hash` from my.telegram.org plus a phone-number login. Use only if you need to act as a *user account* (e.g., reading history a bot can't see). Heavier, and automating user accounts is ToS-sensitive.
3. You can also run a **self-hosted Bot API server** (open-source `telegram-bot-api`, which speaks MTProto upstream): same Bot API surface, but removes file limits (2000 MB uploads, unlimited downloads), allows local file paths, webhooks on any port/IP, and HTTP.

## How the integration works (setup steps)

1. **Create the bot**: message [@BotFather](https://t.me/botfather), send `/newbot`, choose a display name and a username (5–32 chars, must end in `bot`, **cannot be changed later**). BotFather returns the token.
2. **Store the token** as a secret. It can be revoked/regenerated via BotFather (`/revoke`).
3. **Choose update delivery**: for production, `setWebhook` to your HTTPS endpoint with a `secret_token`; for local dev, `getUpdates` long polling (or a tunnel).
4. **Handle updates**: at minimum `message`, `edited_message`, `callback_query`; subscribe explicitly via `allowed_updates` if you need `my_chat_member`, `message_reaction`, `business_message`, etc.
5. **Send replies** with `sendMessage` / `sendPhoto` / `sendDocument`... keyed by `chat_id` (use the numeric user/chat ID from the inbound update).
6. **Map identities**: store `update.message.from.id` (user) and `update.message.chat.id` (conversation) against your helpdesk contact/conversation.

## Auth / token model

- Single static token per bot; passed **in the URL path**: `https://api.telegram.org/bot<token>/METHOD_NAME`. Treat URLs as secrets in logs.
- Requests accept query string, form-encoded, JSON, or multipart (for file upload). Responses: `{"ok": true, "result": ...}` or `{"ok": false, "error_code": ..., "description": ...}`.
- No scopes/permissions on the token — it grants full bot control. Compromise = regenerate via BotFather.

## Receiving messages

### Option A: long polling

```http
GET https://api.telegram.org/bot<token>/getUpdates?timeout=50&offset=<last_update_id + 1>
```

Returns an array of `Update` objects. Simple, great for dev; you must acknowledge by advancing `offset`. **Cannot be used while a webhook is set** (delete with `deleteWebhook` first; check state with `getWebhookInfo`).

### Option B: webhook (recommended for production)

```http
POST https://api.telegram.org/bot<token>/setWebhook
Content-Type: application/json

{
  "url": "https://example.com/webhooks/telegram",
  "secret_token": "a-random-256bit-string_0-9A-Za-z_-",
  "max_connections": 40,
  "allowed_updates": ["message", "edited_message", "callback_query", "my_chat_member"]
}
```

Requirements and behavior:

- **HTTPS only**, on ports **443, 80, 88, or 8443** (any port if self-hosting the Bot API server).
- Self-signed certificates are allowed if you upload the public cert via the `certificate` parameter.
- Telegram POSTs one JSON-serialized `Update` per request and **retries** on non-2xx responses — reply 200 quickly and process async (retries mean you need idempotency on `update_id`).
- **Validation**: there is **no HMAC body signature** (unlike Viber/Meta). Instead, if you set `secret_token` (1–256 chars of `A-Za-z0-9_-`), every webhook request carries it back in the `X-Telegram-Bot-Api-Secret-Token` header. Compare with a constant-time check:

```php
$secret = $request->header('X-Telegram-Bot-Api-Secret-Token');

if (! hash_equals(config('services.telegram.webhook_secret'), (string) $secret)) {
    abort(401, 'Invalid Telegram secret token');
}
```

### Update types (most relevant for a helpdesk)

`message`, `edited_message`, `callback_query` (inline button presses — must be answered with `answerCallbackQuery`), `my_chat_member` (bot blocked/unblocked, added/removed from chats — use to mark contacts unreachable), `chat_member`, `message_reaction`, `channel_post`/`edited_channel_post`, `inline_query`, `chat_join_request`, `business_connection`/`business_message`/`edited_business_message`, payment updates (`pre_checkout_query`, `shipping_query`), `poll_answer`. Newer additions (Bot API 9.x–10.0, 2025–2026) include checklist messages, managed-bot updates and "guest" queries — see changelog; not needed for core messaging.

Example inbound update:

```json
{
  "update_id": 123456789,
  "message": {
    "message_id": 42,
    "from": { "id": 987654321, "is_bot": false, "first_name": "John", "username": "john_doe", "language_code": "en" },
    "chat": { "id": 987654321, "first_name": "John", "type": "private" },
    "date": 1718000000,
    "text": "Hi, I need help with my order"
  }
}
```

## Sending messages

```http
POST https://api.telegram.org/bot<token>/sendMessage
Content-Type: application/json

{
  "chat_id": 987654321,
  "text": "Hi John! An agent will be with you shortly.\n\n<b>Ticket:</b> #1042",
  "parse_mode": "HTML",
  "reply_markup": {
    "inline_keyboard": [[
      { "text": "View ticket", "url": "https://acme.com/t/1042" },
      { "text": "Close ticket", "callback_data": "close:1042" }
    ]]
  }
}
```

- **Formatting**: `parse_mode` = `HTML`, `MarkdownV2`, or legacy `Markdown`. Prefer **HTML** for machine-generated text — MarkdownV2 requires escaping many special characters (`_ * [ ] ( ) ~ \` > # + - = | { } . !`). Alternatively pass raw `entities`.
- **Media**: `sendPhoto`, `sendVideo`, `sendDocument`, `sendAudio`, `sendVoice`, `sendSticker`, `sendLocation`, `sendContact`, `sendMediaGroup` (albums), `sendChatAction` (typing indicator). Media accepted as multipart upload, public URL, or a previously seen `file_id` (re-sending by `file_id` is free and instant).
- **Keyboards**:
  - `InlineKeyboardMarkup` — buttons attached under a message (`url`, `callback_data`, `login_url`, `pay`, `switch_inline_query`). Presses arrive as `callback_query`; you must call `answerCallbackQuery` (optionally with a toast/alert) or the user's client shows a spinner.
  - `ReplyKeyboardMarkup` — replaces the user's keyboard with predefined answer buttons (`request_contact`, `request_location`, etc.); remove with `ReplyKeyboardRemove`.
- **Editing/deleting**: `editMessageText`, `editMessageMedia`, `editMessageReplyMarkup`, `deleteMessage` — useful for live-updating ticket status messages.
- **Files inbound**: an incoming attachment yields a `file_id` → call `getFile` to get a `file_path` → download from `https://api.telegram.org/file/bot<token>/<file_path>` (link valid ≥ 1 hour; ≤ 20 MB on the hosted server).

## Messaging rules & rate limits

- **Bots cannot initiate chats.** A user must start the bot first (`/start`, deep link, or adding it to a group). If the user blocks the bot, sends fail with 403 — mark unreachable (you'll also get `my_chat_member`).
- **No 24-hour window** (unlike WhatsApp/Viber): once started, you may message the user anytime. Free of charge.
- Limits (from the official Bots FAQ): **≤ 1 message/second per chat** (short bursts tolerated), **≤ 20 messages/minute in the same group**, **~30 messages/second globally** for bulk notifications. Exceeding → HTTP 429 with `retry_after` — respect it.
- **Paid broadcasts**: bots that have at least **10,000 Stars balance and 10,000 monthly active users** can enable `allow_paid_broadcast` to send up to **1000 msg/sec**, billed 0.1 Stars per message beyond the free 30/sec.
- **Privacy mode** (groups): on by default. The bot only receives commands addressed to it (`/cmd@your_bot`), replies to its own messages, and service messages — not the whole group conversation. Disable via BotFather `/setprivacy` or make the bot a group admin to receive everything. Irrelevant for 1:1 chats (bots always get all private messages).

## Key features for a multi-channel helpdesk

- **Deep linking**: `https://t.me/your_bot?start=<payload>` (payload ≤ 64 chars of `A-Za-z0-9_-`) delivers `/start <payload>` — use it to tie a Telegram chat to a website visitor/session ID. `?startgroup=` for groups.
- **Telegram Login Widget**: embeddable website login; pair the bot to your domain via BotFather `/setdomain`. The widget returns user data with a hash signed using SHA-256 of the bot token — verify server-side.
- **Telegram Business** (Bot API 7.2+, March 2024, expanded through 9.x): users with a Telegram Business subscription can **connect a bot to their personal account**. The bot receives a `business_connection` update (with `business_connection_id` and `can_reply` permission), then gets `business_message` updates for the chats the owner allows and can reply on the owner's behalf by passing `business_connection_id` to `sendMessage` etc. This lets a helpdesk answer as the business's own Telegram account rather than a bot — worth evaluating as a premium channel.
- **Inline bots**: users invoke the bot via `@botname query` in any chat (must be enabled in BotFather). Niche for helpdesk.
- **Mini Apps (Web Apps)**: full HTML5 UIs launched inside Telegram — an option for embedding a richer support/ticket UI.
- **Bot API 10.0 (May 2026)** added, per the changelog: guest mode (replying in chats the bot isn't a member of), checklists, live photos, managed bots. These are post-research-cutoff features — treat as "verify in changelog before building on them."

## Limitations / gotchas

- No access to a user's **phone number** unless they share it (via `request_contact` button) and no email — identity is the numeric Telegram user ID + optional username.
- No message history API on the Bot API: the bot only sees messages delivered as updates while polling/webhook is configured (the hosted server keeps undelivered updates for ~24h).
- `chat_id` for a private chat equals the user ID, but group/channel IDs are negative numbers — store as 64-bit signed integers.
- Webhook and polling are mutually exclusive; a stale webhook silently breaks polling-based dev setups (`getWebhookInfo` is your friend).
- 20 MB inbound / 50 MB outbound file caps on the hosted API — large customer attachments need the self-hosted Bot API server.
- Markdown escaping bugs are the #1 source of failed sends — prefer HTML parse mode.

## Key Terms

- **@BotFather** — official bot for creating/configuring bots (token, commands, privacy mode, domain, inline mode).
- **Bot token** — `<numeric_bot_id>:<secret>` string; sole credential, embedded in API URLs.
- **Update** — envelope for any inbound event, identified by monotonically increasing `update_id`.
- **`chat_id`** — target for all send methods; user ID for private chats, negative ID for groups/channels.
- **`secret_token`** — value you choose at `setWebhook` time, echoed back in `X-Telegram-Bot-Api-Secret-Token` for webhook authentication.
- **`file_id`** — persistent, reusable handle for any media file seen by the bot; resolve to a download path via `getFile`.
- **Inline keyboard / callback query** — message-attached buttons and the event their presses generate (`answerCallbackQuery` required).
- **Privacy mode** — default group behavior where bots only see commands/replies addressed to them.
- **Deep link** — `t.me/<bot>?start=<payload>` URL that starts the bot with a payload.
- **Business connection** — link between a Telegram Business user account and a bot, letting the bot read/reply in the owner's chats via `business_connection_id`.
- **MTProto / TDLib** — the full client-level API (api_id/api_hash); not needed for bot integrations.
- **Local Bot API server** — self-hosted `telegram-bot-api` binary that lifts file/port limits.

## Official Docs

- Bot API reference: https://core.telegram.org/bots/api
- Bot API changelog: https://core.telegram.org/bots/api-changelog
- Bots overview / features (BotFather, privacy mode, deep linking, Business): https://core.telegram.org/bots/features
- Bots FAQ (rate limits, file limits, broadcasting): https://core.telegram.org/bots/faq
- Webhooks guide: https://core.telegram.org/bots/webhooks
- Telegram Login Widget: https://core.telegram.org/widgets/login
- Self-hosted Bot API server: https://github.com/tdlib/telegram-bot-api
- TDLib (MTProto client API): https://core.telegram.org/tdlib
