# Instagram Platform & Instagram Messaging API

> Knowledge base for engineers integrating Instagram DMs into a multi-channel customer
> messaging product. Verified against Meta's official docs as of June 2026
> (Graph API ~v25.0). Where the docs were ambiguous or unverifiable, it is flagged inline.

## TL;DR

- **Only Instagram professional accounts** (Business or Creator) can be used with the API. Personal accounts cannot.
- There are **two distinct API setups**:
  1. **Instagram API with Facebook Login** — the IG professional account is linked to a Facebook Page; you call `graph.facebook.com` with **Facebook Page access tokens**. Messaging rides on the Messenger Platform ("Messenger API support for Instagram").
  2. **Instagram API with Instagram Login** ("Business Login for Instagram") — no Facebook Page required; you call `graph.instagram.com` with **Instagram User access tokens** and the `instagram_business_*` scopes.
- Messaging is **reply-based**: a user must DM the business first; you then have a **24-hour window** to respond. The **`HUMAN_AGENT` tag** extends this to **7 days** for human (non-automated) replies.
- Users are identified by **IGSID** (Instagram-Scoped ID) — unique per (user, business account) pair; not the public IG user ID.
- Webhooks (`messages`, `messaging_postbacks`, `comments`, …) are delivered through the **Meta App Dashboard Webhooks product** with `"object": "instagram"` payloads.
- The **Instagram Basic Display API is dead** (all requests error since December 4, 2024). Ignore old articles referencing it.

---

## 1. Account types & requirements

Instagram has three account types:

| Type | API access |
|---|---|
| **Personal** | None. Cannot be used with the Instagram Platform APIs. |
| **Professional — Business** | Full API access (messaging, publishing, comments, insights). |
| **Professional — Creator** | Full API access (same surface as Business for messaging purposes). |

Meta's docs state plainly: *"To use the APIs, your app users must have an Instagram professional account."* Converting personal → professional is free and done in the Instagram app (Settings → Account type and tools). Professional accounts are required because the platform's permission model, insights, and messaging policies are built around businesses/creators interacting with consumers — and because Meta retired consumer-grade API access (Basic Display) at the end of 2024.

**App access levels** (both setups):

- **Standard Access** (default): your app can only serve accounts owned by people with roles on the app (developers/testers). Fine for managing your own accounts and for development.
- **Advanced Access**: required when *"your app serves Instagram professional accounts that you don't own or manage"* — i.e. any SaaS like a customer-messaging product. Requires **App Review** plus **Business Verification**.

---

## 2. The two API setups compared

| | **Instagram API with Facebook Login** | **Instagram API with Instagram Login** |
|---|---|---|
| Also known as | Legacy "Instagram Graph API"; messaging = "Messenger API support for Instagram" | "Business Login for Instagram" |
| Facebook Page required | **Yes** — IG professional account must be linked to a FB Page | **No** |
| Login flow | Facebook Login for Business | Business Login for Instagram (OAuth at `instagram.com/oauth/authorize`) |
| Host | `graph.facebook.com` | `graph.instagram.com` (token exchange at `api.instagram.com`) |
| Token type | Facebook **User** and **Page** access tokens (messaging uses Page tokens) | **Instagram User** access tokens (short-lived → long-lived) |
| Messaging permissions | `instagram_basic`, `instagram_manage_messages`, plus Page perms (`pages_manage_metadata`, `pages_show_list`, …) | `instagram_business_basic`, `instagram_business_manage_messages` |
| Comments permissions | `instagram_manage_comments` + `pages_read_engagement` etc. | `instagram_business_manage_comments` |
| Extras only here | **Ads** (CTM/CTD ads), **hashtag search**, **product/collaborator tagging**, marketing messages | — |
| Not available | — | Ads and tagging (*"This API setup cannot access ads or tagging"*) |
| Use when | The business already operates FB Pages / runs ads, or you want one Messenger-style integration covering FB + IG DMs | The business is Instagram-only, or you want the simplest onboarding (no Page linkage) |

For a multi-channel messaging product, supporting **both** is ideal: Facebook Login covers businesses with Pages (and gives you Messenger in the same flow); Instagram Login removes the "you must link a Facebook Page" onboarding cliff that loses IG-only customers.

> Note: the legacy `instagram_basic` / `instagram_content_publish` / `instagram_manage_messages` / `instagram_manage_comments` scope names were **deprecated for the Instagram-Login setup on January 27, 2025** in favor of the `instagram_business_*` names. The non-`business` names still apply on the Facebook-Login path.

---

## 3. Tokens: types, lifetimes, refresh

### 3a. Instagram Login path (Instagram User tokens)

**Step 1 — Authorization** (browser redirect):

```
https://www.instagram.com/oauth/authorize
  ?client_id=<INSTAGRAM_APP_ID>
  &redirect_uri=<REDIRECT_URI>
  &response_type=code
  &scope=instagram_business_basic,instagram_business_manage_messages
  &state=<CSRF_STATE>
```

The returned authorization `code` is valid for **1 hour, single use**.

**Step 2 — Exchange code for a short-lived token** (valid **1 hour**):

```
POST https://api.instagram.com/oauth/access_token
  client_id=<INSTAGRAM_APP_ID>
  client_secret=<INSTAGRAM_APP_SECRET>
  grant_type=authorization_code
  redirect_uri=<REDIRECT_URI>
  code=<CODE>
```

Response includes the short-lived `access_token`, the app user's Instagram-scoped user id, and granted permissions.

**Step 3 — Exchange for a long-lived token** (valid **60 days**, server-side only):

```
GET https://graph.instagram.com/access_token
  ?grant_type=ig_exchange_token
  &client_secret=<INSTAGRAM_APP_SECRET>
  &access_token=<SHORT_LIVED_TOKEN>
```

**Step 4 — Refresh the long-lived token** (returns a fresh 60-day token):

```
GET https://graph.instagram.com/refresh_access_token
  ?grant_type=ig_refresh_token
  &access_token=<LONG_LIVED_TOKEN>
```

Refresh rules: the token must be **at least 24 hours old** and still valid, and the user must have granted `instagram_business_basic`. Tokens **not refreshed within 60 days expire permanently** — the user must re-authenticate. Build a scheduled refresh job (e.g. refresh anything older than ~45 days).

### 3b. Facebook Login path (Page tokens)

- Standard Facebook Login for Business: short-lived **Facebook User token** (~1–2 hours) → exchange at `GET https://graph.facebook.com/<VER>/oauth/access_token?grant_type=fb_exchange_token&client_id=...&client_secret=...&fb_exchange_token=<SHORT_TOKEN>` for a **long-lived User token (~60 days)**.
- From a long-lived User token, fetch the **Page access token** via `GET /me/accounts` (or `/<PAGE_ID>?fields=access_token`). A Page token obtained from a long-lived User token **does not expire** (it's invalidated on password change, permission revocation, etc.).
- Messaging calls are made **with the Page token** against `graph.facebook.com`. The person granting it must be able to perform the `MESSAGE` task on the Page.

---

## 4. Instagram Messaging

### Prerequisites

- Instagram **professional account** (Business or Creator).
- **Facebook Login path:** IG account linked to a FB Page; permissions `instagram_basic`, `instagram_manage_messages`, `pages_manage_metadata` (for webhook subscription).
- **Instagram Login path:** permissions `instagram_business_basic`, `instagram_business_manage_messages`.
- **"Allow access to messages" must be enabled by the account owner in the Instagram app**: Settings → Privacy → Messages → **Connected tools** → toggle *Allow access to messages*. Without this, message webhooks are not delivered and sends fail.
- A verified **webhooks endpoint** subscribed to messaging fields, and the account subscribed to your app (see §5).
- Advanced Access + App Review to serve third-party businesses.

### Identity: IGSID

Conversations don't use usernames or phone numbers. Each consumer gets an **Instagram-Scoped ID (IGSID)** — unique per (user × business account) pair. The same person has a *different* IGSID with every business, and the IGSID differs from their public IG user id. You receive IGSIDs in webhooks; use the **User Profile API** (`GET /<IGSID>?fields=name,username,profile_pic,...`) to enrich, subject to consent/window rules.

### The 24-hour window

*"Only after an Instagram user has sent your app user's Instagram professional account a message can your app send a message to the Instagram user. Your app has 24 hours to respond."* Each new inbound user message (including story replies and ice-breaker taps) re-opens the window. There are **no template/proactive messages** like WhatsApp — outside the window you can only use:

- **`HUMAN_AGENT` tag** — extends the response window to **7 days** after the user's last message, *only* for messages composed by a real human agent. Requires the **Human Agent** permission via App Review. Automated/bot messages under this tag are a policy violation and Meta detects misuse. Example:

```json
POST /<IG_ID or PAGE_ID>/messages
{
  "recipient": {"id": "<IGSID>"},
  "messaging_type": "MESSAGE_TAG",
  "tag": "HUMAN_AGENT",
  "message": {"text": "Following up on your ticket from Friday..."}
}
```

- **Private replies** to comments (see below) — a one-shot opener.
- **Marketing messages** (opt-in based; Facebook-Login path only).

### Send API

- Instagram Login: `POST https://graph.instagram.com/v25.0/<IG_ID>/messages` (or `/me/messages`) with the **Instagram User token** as Bearer.
- Facebook Login: `POST https://graph.facebook.com/v25.0/<PAGE_ID>/messages` (or `/me/messages`) with the **Page token**.

Text example (Instagram Login):

```http
POST https://graph.instagram.com/v25.0/<IG_ID>/messages
Authorization: Bearer <INSTAGRAM_USER_ACCESS_TOKEN>
Content-Type: application/json

{
  "recipient": {"id": "<IGSID>"},
  "message": {"text": "Thanks for reaching out!"}
}
```

Success response: `{"recipient_id": "<IGSID>", "message_id": "<MID>"}`.

### Supported message types

| Type | Notes |
|---|---|
| Text | Max **1000 bytes/characters** |
| Links | Sent as text; render as links |
| Image | png/jpeg, ≤ 8 MB, up to 10 per message |
| Audio | aac/m4a/wav/mp4, ≤ 25 MB |
| Video | mp4/ogg/avi/mov/webm, ≤ 25 MB |
| File | pdf, ≤ 25 MB (Instagram-Login docs) |
| Sticker | heart sticker only (`like_heart`) |
| Reaction | `sender_action: react / unreact` with emoji on a `mid` |
| Media share | share one of the account's **own published posts** (`MEDIA_SHARE`) |
| Reply | `reply_to: {"mid": ...}` to reply to a specific message |
| **Quick replies** | Text-type quick replies, up to 13 (Messenger-style; tap returns a `quick_reply.payload` in the `messages` webhook). *(Max count per Messenger reference; not re-verified on the IG-specific page.)* |
| **Generic template** | Supported: up to **10 elements**, **3 buttons each**, `postback`/`web_url` buttons only. *"Currently not available in the web version"* of Instagram. |
| Product template | Catalog products (Facebook-Login path) |

### Private replies

Reply **privately in DM to a comment** on your posts, reels, ads, or live broadcasts — use `comment_id` as recipient:

```json
POST /<PAGE_ID or IG_ID>/messages
{
  "recipient": {"comment_id": "<COMMENT_ID>"},
  "message": {"text": "Thanks for reaching out, how can I help?"}
}
```

- **One** private reply per comment.
- Within **7 days** of the comment (live broadcasts: only **during** the live).
- A full conversation only starts (24h window opens) if the user responds.
- Permissions on the FB-Login path: `instagram_manage_comments` + `pages_messaging`, Advanced Access.

### Story replies & story mentions

- When a user **replies to your story**, it arrives as a normal `messages` webhook with a `reply_to.story` object (story `id` + asset `url`) and opens the standard 24h window.
- When a user **@mentions you in their story**, you get a message webhook with an attachment of type `story_mention`; you can respond (story-mention reply). The story asset URL is fetchable only while the story is live (24h).

### Ice breakers

FAQ-style conversation starters shown to users opening a new conversation. Up to **4 questions**, set via the Messenger Profile API with `platform=instagram`:

```
POST https://graph.facebook.com/v25.0/me/messenger_profile?platform=instagram
{
  "ice_breakers": [{
    "call_to_actions": [
      {"question": "Where are you located?", "payload": "LOCATION"},
      {"question": "What are your hours?", "payload": "HOURS"}
    ],
    "locale": "default"
  }]
}
```

A tap arrives as a **`messaging_postbacks`** webhook (question as `title`, your `payload` echoed back). A persistent menu is also available on the Facebook-Login path.

### Rate limits (messaging)

From the platform Overview:

- General Graph endpoints: *Calls within 24 hours = 4800 × number of impressions* (Instagram Business Use Case rate limiting).
- **Send API: 100 calls/sec** for text/links; **10 calls/sec** for audio/video sends.
- **Conversations API: 2 calls/sec** per account.
- **Private replies: 750/hour** for posts/reels; 100 calls/sec for live comments.

---

## 5. Webhooks

Instagram webhooks are configured in the **Meta App Dashboard → Webhooks product** (choose the **Instagram** object) — there is no separate delivery system; IG events ride the same app-level webhook infrastructure as Messenger/Pages. Flow:

1. Implement the verification handshake: respond to Meta's `GET` with `hub.challenge` after checking `hub.verify_token`.
2. In the dashboard, subscribe to fields: `messages`, `message_echoes` (Facebook-Login naming; `message_echoes`/echo events let you see agent-sent messages), `messaging_postbacks`, `messaging_reactions`, `messaging_seen`, `messaging_optins`, `messaging_referral`, `messaging_handover`, `comments`, `live_comments`, `mentions`*, `story_insights`, `messaging_policy_enforcement`. (*On the Instagram-Login setup, mention notifications are surfaced through the `comments`/`mentions` fields depending on API path — verify against the field list for your setup.*)
3. **Subscribe the account to your app** — webhooks only fire for subscribed accounts:
   - Instagram Login: `POST graph.instagram.com/<VER>/me/subscribed_apps?subscribed_fields=messages,comments&access_token=<IG_USER_TOKEN>`
   - Facebook Login: `POST graph.facebook.com/<VER>/<PAGE_ID>/subscribed_apps?subscribed_fields=...&access_token=<PAGE_TOKEN>`
4. For messaging events, **"Allow access to messages"** (Connected tools) must be on in the IG app.

Payloads arrive with `"object": "instagram"`. Messaging events use a Messenger-style `messaging` array:

```json
{
  "object": "instagram",
  "entry": [
    {
      "id": "<YOUR_APP_USERS_IG_ID>",
      "time": 1749500000000,
      "messaging": [
        {
          "sender":    {"id": "<IGSID>"},
          "recipient": {"id": "<YOUR_APP_USERS_IG_ID>"},
          "timestamp": 1749499999000,
          "message": {
            "mid": "<MESSAGE_ID>",
            "text": "Hi, is this in stock?"
          }
        }
      ]
    }
  ]
}
```

`messaging_postbacks` (ice breaker / button tap):

```json
{
  "object": "instagram",
  "entry": [{
    "id": "<YOUR_APP_USERS_IG_ID>",
    "time": 1502905976963,
    "messaging": [{
      "sender": {"id": "<IGSID>"},
      "recipient": {"id": "<YOUR_APP_USERS_IG_ID>"},
      "timestamp": 1502905976377,
      "postback": {
        "mid": "<MESSAGE_ID>",
        "title": "<SELECTED_ICEBREAKER_OR_BUTTON_TITLE>",
        "payload": "<YOUR_PAYLOAD>"
      }
    }]
  }]
}
```

`comments` events use the `changes` array form instead (`"field": "comments"`, with comment id, the commenter's IGSID + username, text, media id and `media_product_type`). Echoes of your own sends arrive with an `is_echo` flag — filter them to avoid loops.

---

## 6. Limitations & gotchas

- **No proactive outreach**: no analog of WhatsApp template messages. User must message first; 24h window (7 days with `HUMAN_AGENT`, humans only).
- **No group chats** — 1:1 conversations only.
- **Calls cannot be sent/handled** beyond what the newer "messaging and calling" features expose; classic API surface is DMs only.
- **Message inbox behavior**: messages from non-followers land in the **Requests folder**; *messages in the inactive Requests folder are deleted after 30 days*; messages don't show as "Read" in the IG app until a human reply is sent.
- **Media constraints**: shared posts must be owned by the account; only the heart sticker; generic templates don't render on instagram.com web.
- **Story assets expire**: story-mention/reply media URLs are valid only while the story is live (24h) — download immediately if you need history.
- **Ads & tagging** features are unavailable on the Instagram-Login setup.
- **`Allow access to messages` defaults off** — onboarding must instruct account owners to flip it, or you'll silently get no message webhooks.
- IGSIDs are **per-business**: you cannot correlate the same consumer across two connected IG accounts via IGSID.
- Rate limits differ per endpoint family (see §4); budget for 2 calls/sec on the Conversations API when backfilling threads.

---

## 7. Deprecated/legacy naming — don't get confused by old articles

- **Instagram Basic Display API** — the old read-only API for *personal* accounts (profile + media feed). Deprecated September 4, 2024; **all requests return errors since December 4, 2024**. There is no replacement for personal accounts; users must convert to professional accounts.
- **"Instagram Graph API"** — the old name for what is now **"Instagram API with Facebook Login"** under the umbrella **"Instagram Platform"**. Same API, renamed.
- **"Messenger API support for Instagram" / "Instagram Messaging API"** — messaging for Page-linked IG accounts via the Messenger Platform (`graph.facebook.com`). Still current on the Facebook-Login path; the Instagram-Login path has its own Messaging API on `graph.instagram.com`.
- Old `instagram_*` scopes vs new `instagram_business_*` scopes: the `business` variants are for the **Instagram-Login** setup (mandatory since Jan 27, 2025); the plain variants remain for the **Facebook-Login** setup.

---

## Key Terms

**Professional account**
: An Instagram Business or Creator account. Required for any Instagram Platform API access.

**IGSID (Instagram-Scoped ID)**
: Identifier assigned to a consumer *per business account they message*. Used as `recipient.id` / `sender.id` in messaging. Not the user's global IG id.

**IG ID**
: The app user's (the business's) Instagram professional account id, used in endpoint paths like `/<IG_ID>/messages`.

**Instagram User access token**
: OAuth token from Business Login for Instagram. Short-lived = 1 hour; long-lived = 60 days, refreshable via `refresh_access_token` once ≥24h old.

**Page access token**
: Facebook token tied to a FB Page; used for IG messaging on the Facebook-Login path. Long-lived Page tokens do not expire by time.

**24-hour messaging window**
: Period after the last inbound user message during which the business may send messages of any supported type.

**`HUMAN_AGENT` tag**
: Message tag extending the reply window to 7 days, exclusively for human-composed responses; requires App Review approval.

**Private reply**
: A single DM sent in response to a comment (`recipient.comment_id`), within 7 days of the comment.

**Ice breakers**
: Up to 4 preset FAQ prompts shown when a user opens a new conversation; taps arrive as `messaging_postbacks`.

**Advanced Access**
: App Review-granted level required to serve IG accounts your team doesn't own; needs Business Verification.

**Webhooks object `instagram`**
: All IG events arrive with `"object": "instagram"`; messaging events use a `messaging` array, content events use `changes`.

---

## Official Docs

- Instagram Platform overview: https://developers.facebook.com/docs/instagram-platform/
- Platform Overview (tokens, access levels, rate limits): https://developers.facebook.com/docs/instagram-platform/overview/
- Instagram API with Instagram Login: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/
- Business Login for Instagram (OAuth + token exchange/refresh): https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login
- Messaging API (Instagram Login): https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/messaging-api/
- Conversations API: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/conversations-api/
- Instagram API with Facebook Login: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-facebook-login/
- Messenger Platform — Instagram messaging: https://developers.facebook.com/docs/messenger-platform/instagram
- Send a Message (FB Login path): https://developers.facebook.com/docs/messenger-platform/instagram/features/send-message/
- Private Replies: https://developers.facebook.com/docs/messenger-platform/instagram/features/private-replies/
- Ice Breakers: https://developers.facebook.com/docs/messenger-platform/instagram/features/ice-breakers/
- Generic Template: https://developers.facebook.com/docs/messenger-platform/instagram/features/generic-template/
- Human Agent Escalation: https://developers.facebook.com/docs/messenger-platform/instagram/features/human-agent-escalation/
- Webhooks: https://developers.facebook.com/docs/instagram-platform/webhooks
- Webhook payload examples: https://developers.facebook.com/docs/instagram-platform/webhooks/examples
- API Reference: https://developers.facebook.com/docs/instagram-platform/reference/
