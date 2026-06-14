# Messaging Platforms Knowledge Base

Reference docs for integrating customer messaging channels: Meta (Messenger, Instagram, WhatsApp), Viber, and Telegram. Researched against the official developer docs in June 2026 — anything uncertain is flagged inline with a "verify" note in the individual files.

## Structure

```
16-livecaller-hub-integrations/
├── README.md            ← this file (index + cross-platform comparison)
├── GLOSSARY.md          ← cross-platform terms & definitions
├── meta/                ← everything on developers.facebook.com
│   ├── overview.md      ← Meta apps, app modes, App Review, Business Verification
│   ├── tokens.md        ← all access-token types, lifetimes, exchange flows, debugging
│   ├── permissions.md   ← permission/scope model, Standard vs Advanced Access
│   ├── graph-api.md     ← versioning, rate limits, error codes, batch requests
│   ├── webhooks.md      ← webhook handshake, signature validation, retries
│   ├── messenger.md     ← Messenger Platform (Send API, 24h window, handover)
│   ├── instagram.md     ← Instagram messaging (two login paths, IGSID, webhooks)
│   └── whatsapp.md      ← WhatsApp Cloud API (WABA, templates, pricing, limits)
├── viber/
│   └── viber.md         ← Viber Bot API (static token, mandatory webhook, sessions)
└── telegram/
    └── telegram.md      ← Telegram Bot API (BotFather token, polling vs webhook)
```

## Where to start

- New to Meta integrations? Read [meta/overview.md](meta/overview.md) → [meta/tokens.md](meta/tokens.md) → [meta/webhooks.md](meta/webhooks.md), then the channel doc you need. The token model is shared across Messenger, Instagram, and WhatsApp — understand it once.
- Viber and Telegram are self-contained: each platform doc covers auth, webhooks, and sending in one file.

## Cross-platform comparison

| | Messenger | Instagram | WhatsApp Cloud API | Viber | Telegram |
|---|---|---|---|---|---|
| **Credential** | Page Access Token (Meta app + Page) | Page token or IG User token (two login paths) | System User token (WABA + phone number) | Static bot auth token | Static bot token (BotFather) |
| **OAuth involved** | Yes (Facebook Login) | Yes (FB or Instagram Login) | Yes (Embedded Signup for tech providers) | No | No |
| **App review needed** | Yes (Advanced Access) | Yes | Yes (+ Business Verification) | No | No |
| **Receive messages** | Webhook (verify token + signature) | Webhook (same Meta app) | Webhook (per WABA) | Webhook (mandatory, HMAC signature) | Webhook **or** long polling (`getUpdates`) |
| **User identifier** | PSID (page-scoped) | IGSID (account-scoped) | Phone number / WA ID | User ID (bot-scoped) | `chat_id` (global per user) |
| **Who messages first** | User must message the Page first | User must message the account first | Business may initiate with approved template | Bot can message subscribed users | User must `/start` the bot first |
| **Reply window** | 24 h (then `HUMAN_AGENT` tag, 7 days) | 24 h (then `HUMAN_AGENT` tag, 7 days) | 24 h customer-service window (then paid template) | 24 h / 60-message free session, then billed | None — open chats are unrestricted |
| **Paid messaging** | No | No | Yes (per-message template pricing since Jul 2025) | Yes (commercial model since 2024, ~€100/mo + per-msg) | No (except paid broadcasts >30 msg/s) |

## Maintenance notes

- Meta Graph API was at **v25.0** at research time; versions live ~2 years. Re-check version pins when revisiting.
- Meta is migrating docs from `developers.facebook.com/docs/...` to a new JS-rendered `developers.facebook.com/documentation/...` tree — old deep links may move.
- WhatsApp pricing changed from per-conversation to **per-message** on July 1, 2025; re-verify rates before any pricing-sensitive work.
- Messenger message tags (except `HUMAN_AGENT`) were deprecated Jan 2026 — older blog posts and SDKs still reference them.
