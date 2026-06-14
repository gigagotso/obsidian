# Glossary — Cross-Platform Terms & Definitions

Terms you'll meet across the messaging-platform docs. Platform-specific deep dives live in each platform's file; this is the shared vocabulary.

## Meta platform (Facebook / Instagram / WhatsApp)

**Meta App** — A container on developers.facebook.com that holds your integration's credentials, permissions, webhook subscriptions, and review status. One app can serve Messenger, Instagram, and WhatsApp simultaneously.

**App ID / App Secret** — The app's public identifier and its confidential key. The secret signs `appsecret_proof` and is required for token exchanges. Never ships to clients.

**App Mode (Development / Live)** — In Development mode the app only works for users with a role on the app (admin/developer/tester). Live mode requires App Review for the permissions you use.

**App Review** — Meta's approval process for Advanced Access to permissions. You submit screencasts and usage descriptions; required before serving real users.

**Business Verification** — Verification of your company's legal identity in Meta Business Manager. Required for Advanced Access to business permissions and for WhatsApp production access.

**Business Portfolio (Business Manager)** — The Meta Business account that owns Pages, WhatsApp Business Accounts, ad accounts, and system users.

**Graph API** — Meta's versioned HTTP API (`graph.facebook.com/vXX.X/...`) underlying all three channels. Versions live ~2 years after release.

**Permission / Scope** — A grant a user gives your app during OAuth (e.g. `pages_messaging`, `instagram_manage_messages`, `whatsapp_business_messaging`). Standard Access works for app-role users; Advanced Access (via App Review) works for everyone.

### Tokens

**User Access Token** — Obtained via Facebook Login. Short-lived (~1–2 h) by default; exchangeable for a long-lived token (~60 days). Used to fetch Page tokens.

**Page Access Token** — Acts as a Facebook Page; the credential for Messenger (and Instagram via Facebook Login). A Page token derived from a long-lived user token does not expire (but invalidates if the user changes password, loses Page role, etc.).

**System User Access Token** — Issued to a non-human "system user" inside a Business Portfolio. Can be created with no expiry. The recommended production credential for WhatsApp and server-to-server integrations.

**App Access Token** — `app_id|app_secret` (or via OAuth endpoint). Authenticates the app itself — used for webhook subscription management and `/debug_token`.

**Client Token** — A low-privilege token embeddable in mobile/web clients; rarely used in server integrations.

**Token exchange** — Trading a short-lived token for a long-lived one via `oauth/access_token?grant_type=fb_exchange_token`.

**`/debug_token`** — Graph endpoint (and the Access Token Debugger UI) that shows a token's type, scopes, expiry, and validity.

**`appsecret_proof`** — `HMAC-SHA256(access_token, app_secret)` sent alongside API calls; lets you require that calls carrying your tokens also prove possession of the app secret.

**Embedded Signup** — Meta's pop-up onboarding flow that lets a Tech Provider onboard a customer's WhatsApp Business Account and obtain a business integration system-user token in minutes.

### Messaging concepts

**PSID (Page-Scoped ID)** — A user's ID specific to one Page in Messenger. The same person has different PSIDs on different Pages. Not interchangeable with ASID.

**ASID (App-Scoped ID)** — A user's ID specific to one app (from Facebook Login). Different from PSID; mapping between them requires special APIs.

**IGSID (Instagram-Scoped ID)** — The Instagram equivalent of a PSID: a user's ID scoped to one Instagram professional account.

**WABA (WhatsApp Business Account)** — The container for WhatsApp phone numbers, message templates, and webhook subscriptions, owned by a Business Portfolio.

**Phone Number ID** — The Cloud API identifier for a registered WhatsApp business number; the `/<PHONE_NUMBER_ID>/messages` endpoint sends from it.

**24-hour window (standard messaging window)** — On Messenger, Instagram, and WhatsApp, a business may send free-form messages only within 24 h of the user's last message. Outside it: `HUMAN_AGENT` tag (Messenger/Instagram, 7 days) or a paid template message (WhatsApp).

**Message Tag** — A label permitting specific out-of-window Messenger/Instagram messages. As of Jan 2026 only `HUMAN_AGENT` remains supported.

**Template Message (WhatsApp)** — A pre-approved message format (categories: marketing, utility, authentication) required to initiate conversations or message outside the 24 h window. Billed per message since July 2025.

**Handover Protocol** — Messenger/Instagram mechanism letting two apps (e.g. a bot and a human-agent inbox) share one Page's conversations via primary/secondary receiver roles.

**Webhook Verify Token** — A string you choose; Meta echoes it in the `hub.challenge` GET handshake when you register a webhook callback URL.

**`X-Hub-Signature-256`** — Header on Meta webhook POSTs: HMAC-SHA256 of the raw body keyed with your app secret. Always validate it.

**Quality Rating / Messaging Limits (WhatsApp)** — Per-number health score and the tiered cap on business-initiated conversations per 24 h (250 → 1K → 10K → 100K → unlimited), scaling with volume and quality.

## Viber

**Bot Auth Token** — The single static credential for a Viber bot, sent as the `X-Viber-Auth-Token` header. No OAuth.

**`set_webhook`** — Registers your callback URL. Viber has no polling — a webhook is mandatory to receive anything.

**`X-Viber-Content-Signature`** — HMAC-SHA256 of the payload keyed with the bot token; validate on every webhook.

**Subscribed user** — A user who pressed Subscribe (or messaged the bot, depending on flow). `conversation_started` alone is *not* a subscription. User IDs are scoped per bot.

**Commercial model / Sessions** — Since 2024 Viber bots are paid in most countries: a monthly fee plus billed chatbot-initiated messages; a user-initiated session grants a free 24 h / 60-message reply window.

## Telegram

**Bot Token** — `123456:ABC-DEF...` issued by @BotFather; the only credential. No OAuth.

**Bot API vs MTProto** — The Bot API (`api.telegram.org/bot<token>/...`) is the simple HTTPS layer for bots. MTProto (with `api_id`/`api_hash`, via TDLib) is the full client protocol for user-account automation. Helpdesk bots use the Bot API.

**`getUpdates` vs `setWebhook`** — The two mutually exclusive ways to receive updates: long polling vs HTTPS webhook (ports 443/80/88/8443).

**`secret_token`** — Optional string set with `setWebhook`; Telegram echoes it in the `X-Telegram-Bot-Api-Secret-Token` header so you can authenticate webhook calls (Telegram has no payload signature).

**`chat_id`** — The identifier you send to; a user's ID is global (same across bots), unlike Meta's page-scoped IDs.

**`/start` requirement** — Bots cannot initiate chats; a user must message the bot (or hit a `t.me/bot?start=payload` deep link) first.

**Privacy mode** — Group setting controlling whether a bot sees all group messages or only commands/replies.

**`file_id`** — Reusable, bot-scoped handle for any uploaded media; download via `getFile` (20 MB limit via hosted Bot API; 50 MB upload).
