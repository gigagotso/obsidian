# Facebook Messenger Platform — Integration Knowledge Base

> **Audience:** engineers building a multi-channel customer messaging product (live chat / helpdesk).
> **Last verified:** 2026-06-10 against `developers.facebook.com` (Messenger Platform docs are migrating from `/docs/messenger-platform/...` to `/documentation/business-messaging/messenger-platform/...`; both URL trees currently resolve, but some legacy pages now 404).
> Items marked **[memory — verify]** could not be re-confirmed verbatim from the live docs and should be re-checked before relying on exact field names.

---

## TL;DR

- A Messenger integration = **Meta App** (with the Messenger product) connected to a **Facebook Page**. You send messages with a **Page Access Token** via the Graph API **Send API** (`POST /<PAGE_ID>/messages` or `/me/messages`) and receive events via **Webhooks**.
- Users are identified by **PSID** (Page-Scoped ID) — stable per user *per Page*, not globally.
- **You cannot message a user first.** The user must initiate (message, m.me link, plugin, ad click). That opens a **24-hour standard messaging window**; inside it you can send anything (including promo). Outside it you need the **HUMAN_AGENT tag (7 days)**, a pre-approved **Utility Template**, or the **Marketing Messages API** (opt-in based).
- **Big 2026 change:** the legacy message tags `ACCOUNT_UPDATE`, `CONFIRMED_EVENT_UPDATE`, `POST_PURCHASE_UPDATE` were deprecated 2026-01-12 and **hard-rejected (error code 100) since 2026-04-27**. `HUMAN_AGENT` remains and is the tag that matters for helpdesk products. Recurring Notifications ("marketing messages") were also shut down 2026-02-10 in favor of the new Marketing Messages API (MAPI).
- For bot + human-agent coexistence on the same Page, use the **Handover Protocol** (primary/secondary receiver apps, `pass_thread_control`, `standby` webhook). Meta's Page Inbox can act as the secondary receiver (app ID `263902037430900`).
- Rate limits: roughly `200 × engaged users` calls per 24h, with per-Page Send API throughput caps (~300 msg/s text). Webhooks must return `200 OK` within 20 seconds.

---

## 1. How a Messenger Integration Works End-to-End

### Architecture

```
User (Messenger app)  ⇄  Meta / Facebook Page
                              │
              webhook events  │  Send API calls
              (HTTPS POST)    │  (Graph API, Page Access Token)
                              ▼
                     Your app (helpdesk backend)
```

- Every business presence on Messenger is a **Facebook Page** (or Instagram Professional account for IG messaging). There is no "bot account" — your app acts *on behalf of a Page*.
- Your **Meta App** (created in the App Dashboard) holds the webhook subscription and the permissions; each customer connects their Page to your app via Facebook Login for Business / the Page-token flow.
- All Send API calls are authenticated with a **Page Access Token**: "All requests to Messenger Platform APIs are authenticated by including a page-level access token in the `access_token` parameter."

### Setup steps (numbered)

1. **Create / publish the Facebook Page.** The Page must be *published* before App Review.
2. **Create a Meta App** at developers.facebook.com (Business type) and add the **Messenger** product in the App Dashboard.
3. **Get a Page Access Token.** In Messenger settings, link the Page and generate a token (App Dashboard token-generation UI does not store tokens — each selection generates a new one). In production, multi-tenant products obtain a **long-lived Page token** via OAuth: user grants permissions → exchange short-lived user token for long-lived user token → `GET /me/accounts` returns Page tokens (Page tokens derived from long-lived user tokens do not expire by time, but are invalidated on password change / permission revocation).
4. **Required permissions** (granted by the Page admin during OAuth, approved via App Review for public use):
   - `pages_messaging` — send/manage Page conversations via Messenger (the core one).
   - `pages_manage_metadata` — required dependency; needed to subscribe the app to the Page's webhooks and manage settings.
   - `pages_show_list` — required dependency; list the Pages the user administers (used in your "connect your Page" picker).
   - `human_agent` — separate permission for the HUMAN_AGENT tag (7-day human-reply window). Requires its own App Review approval. **[memory — the human_agent permission reference page now 404s; the 7-day behavior is confirmed in the live policy docs]**
   - `pages_read_engagement` is commonly also needed to read Page content/comments (e.g., for private replies via the `feed` webhook). **[memory — verify for your exact use case]**
   - Instagram messaging additionally needs `instagram_basic` + `instagram_manage_messages`.
5. **Set up the webhook endpoint** (HTTPS, valid cert): handle the GET verification request (`hub.mode`, `hub.verify_token`, `hub.challenge`) and POST event notifications; validate `X-Hub-Signature-256`.
6. **Subscribe the app to the Page** — `POST /<PAGE_ID>/subscribed_apps` with the Page token and `subscribed_fields` (e.g. `messages,messaging_postbacks,messaging_optins,message_deliveries,message_reads,messaging_referrals,messaging_handovers,standby`).
7. **Test in development.** Before approval, "Page tokens only allow your Page to interact with Facebook accounts that have been granted the Administrator, Developer, or Tester role for your app." Add testers via the app's Roles tab.
8. **App Review** for Advanced Access to `pages_messaging` (and `human_agent`). Requirements per the official docs:
   - Comply with platform policies and Community Standards.
   - The associated Page must be published.
   - "Ensure your webhook is functioning as expected and returning a `200 OK` response to webhook events within 20 seconds."
   - Provide a screencast demonstrating message sending (including generating the API request) and test credentials for gated functionality.
   - **Business Verification** of your company is required for Advanced Access as a tech provider.
9. **Go live:** switch the app to Live mode; now any user who messages a connected Page reaches your product.

---

## 2. Send API

### Endpoint

```
POST https://graph.facebook.com/v25.0/<PAGE_ID>/messages?access_token=<PAGE_ACCESS_TOKEN>
```

- `/me/messages` also works (`me` resolves to the Page the token belongs to). Docs examples use `/<PAGE_ID>/messages`.
- Graph API **v25.0** is current as of mid-2026 (released Feb 2026); pin a recent version, don't hardcode old ones (v19 dies 2026-05-21, v20 dies 2026-09-24).

### Request body parameters

| Param | Notes |
|---|---|
| `recipient` | `{"id": "<PSID>"}` in almost all cases. Alternatives: `{"user_ref": ...}` (checkbox plugin), `{"comment_id": ...}` / `{"post_id": ...}` (private replies), `{"notification_messages_token": ...}` (legacy marketing messages). |
| `messaging_type` | `RESPONSE` (reply within 24h window), `UPDATE` (non-promotional update within window), `MESSAGE_TAG` (outside window, requires `tag`). |
| `message` | `text`, or `attachment`, plus optional `quick_replies`, `metadata`. Mutually exclusive with `sender_action`. |
| `sender_action` | `typing_on`, `typing_off`, `mark_seen` (sent *instead of* `message`). |
| `tag` | Required when `messaging_type=MESSAGE_TAG`. As of 2026 effectively only `HUMAN_AGENT` (see §4). |
| `notification_type` | `REGULAR` (default), `SILENT_PUSH`, `NO_PUSH`. **[memory — verify]** |
| `persona_id` | Send as a named/avatared persona (Personas API) so users can see which human agent replied. |

Successful response returns `{"recipient_id": "<PSID>", "message_id": "<MID>"}`.

### Examples

**Text message (reply inside 24h window)**

```bash
curl -X POST "https://graph.facebook.com/v25.0/me/messages?access_token=<PAGE_ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "recipient": {"id": "<PSID>"},
    "messaging_type": "RESPONSE",
    "message": {"text": "Hi! How can we help today?"}
  }'
```

**Attachment (image / audio / video / file by URL)**

```json
{
  "recipient": {"id": "<PSID>"},
  "messaging_type": "RESPONSE",
  "message": {
    "attachment": {
      "type": "image",
      "payload": {"url": "https://example.com/receipt.png", "is_reusable": true}
    }
  }
}
```

Reusable attachments can be pre-uploaded via the Attachment Upload API (`POST /me/message_attachments`) to get an `attachment_id`. **[memory — verify endpoint name]**

**Quick replies** (max **13** per message; content types `text`, `user_phone_number`, `user_email`)

```json
{
  "recipient": {"id": "<PSID>"},
  "messaging_type": "RESPONSE",
  "message": {
    "text": "Pick a color:",
    "quick_replies": [
      {
        "content_type": "text",
        "title": "Red",
        "payload": "<POSTBACK_PAYLOAD>",
        "image_url": "http://example.com/img/red.png"
      },
      {"content_type": "text", "title": "Green", "payload": "PICK_GREEN"}
    ]
  }
}
```

When tapped, the next `messages` webhook contains `message.quick_reply.payload` plus `message.text` = the button title.

**Templates** — structured messages sent as `attachment.type: "template"`:

| Template | Purpose |
|---|---|
| `button` | Text + up to **3** buttons (`web_url`, `postback`, `phone_number`). |
| `generic` | Card/carousel: title, subtitle, image, default_action, up to 3 buttons per element (up to 10 elements horizontally scrollable). |
| `media` | Image/GIF/video with optional CTA button. |
| `receipt` | Order confirmation: summary, payment details, shipping. |
| `product` | Renders items from a connected catalog. |
| `coupon` | Structured discount/offer. |
| `customer_feedback` | Native CSAT/feedback survey. |
| Utility templates | Pre-approved templates for order/account/appointment updates outside the 24h window (the replacement for the deprecated message tags). |

Generic template example:

```json
{
  "recipient": {"id": "<PSID>"},
  "messaging_type": "RESPONSE",
  "message": {
    "attachment": {
      "type": "template",
      "payload": {
        "template_type": "generic",
        "elements": [{
          "title": "Order #1042",
          "subtitle": "2 items — $54.00",
          "image_url": "https://example.com/order.png",
          "buttons": [
            {"type": "web_url", "url": "https://example.com/orders/1042", "title": "View order"},
            {"type": "postback", "title": "Talk to an agent", "payload": "HANDOFF_TO_AGENT"}
          ]
        }]
      }
    }
  }
}
```

**Sender actions** (great for helpdesk UX — fire `mark_seen` when an agent opens the conversation, `typing_on` while composing):

```json
{
  "recipient": {"id": "<PSID>"},
  "sender_action": "typing_on"
}
```

Docs guidance: send `mark_seen` on receipt so users don't feel ignored; keep the `typing_on` → message gap natural.

---

## 3. Webhooks

### Setup & verification

1. Expose an HTTPS endpoint. Meta first sends a **GET verification request**: query params `hub.mode=subscribe`, `hub.verify_token=<your token>`, `hub.challenge`. Check the verify token matches, then echo back `hub.challenge` with `200`.
2. Configure the callback URL + verify token in App Dashboard → Webhooks (object type **Page**), subscribe to fields.
3. Subscribe each connected Page: `POST /<PAGE_ID>/subscribed_apps?subscribed_fields=messages,messaging_postbacks,...` with the Page token.
4. **Verify payload integrity** on every POST: `X-Hub-Signature-256` header = HMAC-SHA256 of the raw body keyed with your **App Secret** (`sha256=<hex>`).
5. Respond `200 OK` within **20 seconds**; process asynchronously. Meta retries failed deliveries with backoff and can disable the subscription if your endpoint keeps failing.

Note (Graph v25 changelog): webhook **mTLS certificates change to the Meta CA by 2026-03-31** — relevant if you pin client certs for webhook origin verification.

### Webhook fields (Page object) relevant to messaging

| Field | Fires when |
|---|---|
| `messages` | User sends a message (text, attachment, quick-reply tap, reply). |
| `messaging_postbacks` | User taps a postback button, Get Started, persistent-menu item, or ice breaker. |
| `messaging_optins` | Opt-in events: checkbox / "Send to Messenger" plugin, notification-messages opt-in (token delivered here). |
| `message_deliveries` | Messages you sent were delivered (`delivery.mids`, `delivery.watermark`). |
| `message_reads` | User read messages up to `read.watermark`. |
| `messaging_referrals` | User enters via m.me link with `ref`, ad, or QR code (existing thread). |
| `message_echoes` | Echo of every message *your Page* sent (incl. from Page Inbox / other apps) — key for syncing agent replies made elsewhere. |
| `message_reactions` | User reacts/unreacts to a message. |
| `message_edits` | User edits a sent message. |
| `messaging_handovers` | Thread control changed (handover protocol, see §5). |
| `standby` | Message activity in a thread your app does **not** currently control. |
| `messaging_account_linking` | Account linking events. |
| `response_feedback`, `send_cart`, `messaging_game_plays` | Feedback surveys, commerce carts, instant games. |
| `feed` (Page webhook, not Messenger-specific) | Comments/posts on the Page — used to trigger **Private Replies** (`recipient: {"comment_id": ...}`) that open a Messenger conversation from a comment. |

### Payload structure

Every POST has the envelope:

```json
{
  "object": "page",
  "entry": [
    {
      "id": "<PAGE_ID>",
      "time": 1741234567890,
      "messaging": [ { ...one event... } ]
    }
  ]
}
```

(Batching: an `entry` may contain multiple events; iterate defensively. Instagram messaging arrives with `"object": "instagram"`.)

**`messages` event — text:**

```json
{
  "sender": {"id": "<PSID>"},
  "recipient": {"id": "<PAGE_ID>"},
  "timestamp": 1741234567890,
  "message": {
    "mid": "m_AbCdEf...",
    "text": "Where is my order?"
  }
}
```

**`messages` event — attachment:** `message.attachments[]` with `type` ∈ `image, audio, video, file, sticker, reel, ig_reel, post, ig_post, appointment_booking` and `payload.url`.

**`messages` event — quick reply tapped:**

```json
{
  "message": {
    "mid": "m_...",
    "text": "Green",
    "quick_reply": {"payload": "DEVELOPER_DEFINED_PAYLOAD_FOR_PICKING_GREEN"}
  }
}
```

**`messaging_postbacks` event:**

```json
{
  "sender": {"id": "<PSID>"},
  "recipient": {"id": "<PAGE_ID>"},
  "timestamp": 1527459824,
  "postback": {
    "mid": "m_MESSAGE-ID",
    "title": "TITLE-FOR-THE-CTA",
    "payload": "USER-DEFINED-PAYLOAD",
    "referral": {
      "ref": "USER-DEFINED-REFERRAL-PARAM",
      "source": "SHORT-URL",
      "type": "OPEN_THREAD"
    }
  }
}
```

**`messaging_referrals` event (m.me link with ref):**

```json
{
  "sender": {"id": "<PSID>"},
  "recipient": {"id": "<PAGE_ID>"},
  "timestamp": 1458692752478,
  "referral": {
    "ref": "REF-PARAMETER-INFORMATION",
    "source": "SHORTLINK",
    "type": "OPEN_THREAD"
  }
}
```

**`messaging_optins` event (notification messages opt-in):**

```json
{
  "sender": {"id": "<PSID>"},
  "recipient": {"id": "<PAGE_ID>"},
  "timestamp": "<TIMESTAMP>",
  "optin": {
    "type": "notification_messages",
    "payload": "ADDITIONAL-INFORMATION",
    "notification_messages_token": "<TOKEN>",
    "notification_messages_timezone": "<TIMEZONE-ID>",
    "token_expiry_timestamp": "<TIMESTAMP>",
    "user_token_status": "<TOKEN-STATUS>",
    "notification_messages_status": "<STATUS>",
    "title": "<TITLE>"
  }
}
```

**`message_deliveries` / `message_reads`:** carry a `watermark` (epoch ms) — everything sent before it is delivered/read. **[memory — exact shape: `"delivery": {"mids": [...], "watermark": ...}`, `"read": {"watermark": ...}`]**

---

## 4. Messaging Window & Message Tags

### The 24-hour standard messaging window

- "Businesses have up to 24 hours to respond to a user. Messages sent within the 24-hour window may contain promotional content."
- The window **opens/refreshes** on user actions: sending a message, tapping a CTA/postback/quick reply, reacting, clicking an m.me/ig.me link, or messaging via a plugin or click-to-Messenger ad.
- Outside the window you may send only: a tagged message (`MESSAGE_TAG` + allowed tag), a pre-approved **Utility Template**, **Marketing Messages API** sends (opt-in), or **Sponsored Messages** (paid).

### Message tags — state as of June 2026

| Tag | Status | Window / rules |
|---|---|---|
| `HUMAN_AGENT` | **Active — the important one for helpdesks** | "Allows businesses to manually respond to user messages within a 7-day period." Human agents only — no automated/bot messages. Requires the `human_agent` permission via App Review. |
| `ACCOUNT_UPDATE` | **Deprecated** (announced 2026-01-12) | Since **2026-04-27** API requests using it get **error code 100**. Migrate to Utility Templates. |
| `CONFIRMED_EVENT_UPDATE` | **Deprecated** | Same as above. |
| `POST_PURCHASE_UPDATE` | **Deprecated** | Same as above. |
| `CUSTOMER_FEEDBACK` | Phased out alongside the above; use the customer_feedback template within the window. **[memory — verify]** | — |

**Sending with HUMAN_AGENT:**

```json
{
  "recipient": {"id": "<PSID>"},
  "messaging_type": "MESSAGE_TAG",
  "tag": "HUMAN_AGENT",
  "message": {"text": "Hi, this is Anna from support following up on your ticket."}
}
```

Helpdesk guidance: treat the conversation as (a) freely repliable for 24h after the user's last message, (b) repliable by *human agents only* for 7 days (send with `HUMAN_AGENT`), (c) closed after that — you need the user to come back, a utility template, or an opted-in marketing message. Track the window per conversation from the last inbound `messages` event timestamp.

### Opt-in / re-engagement mechanisms

- **One-Time Notification (OTN)** — legacy beta ("notify me once" button → `one_time_notif_token` for one message). The docs page has been removed and the feature is effectively obsolete in 2026; do not build on it.
- **Recurring Notifications / "marketing messages"** — **deprecated 2026-02-10**. Tokens, `notification_messages` template and related webhooks are being switched off.
- **Marketing Messages API (MAPI)** — the replacement; "launched to all tech providers on July 1, 2025." Subscription-token based, currently limited to approved regions (AU, BR, CL, CO, HK, IN, ID, IL, MY, MX, NZ, PE, PH, SA, SG, TW) and routed through tech providers. If your product needs proactive marketing sends, integrate MAPI; for transactional updates use Utility Templates.

### Compliance notes

- **Automated experience disclosure:** "Automated chat experiences must disclose that a person is interacting with an automated service: at the beginning of any conversation or message thread, after a significant lapse of time, or when a chat moves from human interaction to automated experience" (required in certain jurisdictions, e.g. California; safest to always disclose).
- No promotional content outside the window without an approved mechanism. Tag misuse is a common cause of Page messaging restrictions.

---

## 5. Handover Protocol (bot + human agent coexistence)

The Handover Protocol (docs now title it **"Conversation Routing"**) lets **two or more apps share one Page's Messenger conversations** — the canonical helpdesk pattern: your bot/automation app and a human-agent app (your own inbox or Meta's Page Inbox).

### Roles

- **Primary receiver**: the app that receives `messages` webhooks and owns threads by default. Configured by the Page admin in **Page Settings → Messaging / Messenger receivers** (App Dashboard "App roles" historically). Only one primary.
- **Secondary receiver(s)**: receive `standby` webhooks for threads they don't control; can request or be passed control.
- **Page Inbox** (the inbox in Meta Business Suite) can be set as a receiver so human agents can reply natively — its app ID is **`263902037430900`**.

### Thread-control endpoints (Page token, `POST` unless noted)

| Endpoint | Who calls it | Effect |
|---|---|---|
| `/<PAGE_ID>/pass_thread_control` | Current thread owner | Hand the thread to `target_app_id` (e.g. bot → human inbox on "talk to agent"). |
| `/<PAGE_ID>/request_thread_control` | Secondary receiver | Ask the owner for control; owner gets a `messaging_handovers` event and should pass control. |
| `/<PAGE_ID>/take_thread_control` | Primary receiver only | Forcibly take control back from a secondary app. |
| `/<PAGE_ID>/release_thread_control` | Current owner | Release control back to the primary receiver. |
| `GET /<PAGE_ID>/secondary_receivers?fields=id,name` | Primary | List secondary receiver apps. |
| `GET /<PAGE_ID>/thread_owner?recipient=<PSID>` | Any receiver | Which app currently owns the thread. |

**[memory — endpoint names corroborated by the live "Conversation Routing" doc; re-verify exact request shape]** Example:

```bash
curl -X POST "https://graph.facebook.com/v25.0/me/pass_thread_control?access_token=<PAGE_ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "recipient": {"id": "<PSID>"},
    "target_app_id": 263902037430900,
    "metadata": "user requested human agent"
  }'
```

### Webhooks involved

- `messaging_handovers` — "the ownership of a conversation has changed from one app to another" (sub-events: `pass_thread_control`, `take_thread_control`, `request_thread_control`, `app_roles`).
- `standby` — "a Page receives a message but the app is not the current owner of the conversation." Lets the non-owning app (e.g. your analytics or your bot) observe the thread. Apps in standby cannot use the normal Send API on that thread.
- Subscribe both receiver apps to `messages`, `messaging_postbacks`, `messaging_handovers`, `standby`.

### Typical helpdesk flow

1. Your app is **primary**; bot answers FAQs.
2. User taps "Talk to an agent" (postback) → you route internally, or `pass_thread_control` to the human-agent app / Page Inbox.
3. While agents work the thread, the bot app receives `standby` events (stay silent).
4. Agent closes the ticket → `release_thread_control` (or `take_thread_control` from the primary) returns the thread to the bot.
5. Use `message_echoes` to mirror Page-Inbox/agent messages into your own transcript.

If your product is the *only* app on the Page, you don't strictly need handover — but enabling it (and playing nice with Page Inbox as secondary) avoids the classic "bot swallows messages that humans should see" complaint.

---

## 6. Conversation entry points & profile features

- **m.me links:** `https://m.me/<PAGE_USERNAME>` opens the thread; `https://m.me/<PAGE_USERNAME>?ref=<REF>` delivers the ref via `messaging_referrals` (existing thread) or on the `postback` referral of Get Started (new thread). Use refs to attribute website/QR/email entry points.
- **Get Started button & welcome screen:** set `get_started: {"payload": "..."}` on the Messenger Profile; first interaction arrives as `messaging_postbacks`. `greeting` sets pre-conversation welcome text.
- **Ice breakers:** up to **4** FAQ prompts shown to new users; set via Messenger Profile API with the localized format (`ice_breakers: [{"call_to_actions": [{"question","payload"}], "locale": "default"}]`); taps arrive as postbacks. Not deprecated, but the old non-localized format is being retired.
- **Persistent menu:** always-visible menu; set via `POST /me/messenger_profile` with `persistent_menu`. Up to 20 items in `call_to_actions` (button types `web_url` and `postback` only; title ≤ 30 chars; postback payload ≤ 1000 chars); supports `locale` variants and `composer_input_disabled`.
- **Chat Plugin** (website embed), **Checkbox plugin** (`user_ref` opt-in), and **click-to-Messenger ads** are the other standard entry points.

---

## 7. Identity: PSID vs ASID vs IGSID

- **PSID (Page-Scoped ID):** how a user appears to a *Page* in Messenger. The `sender.id` in webhooks and `recipient.id` in Send API. **Different for the same human on each Page.**
- **ASID (App-Scoped ID):** how the same user appears to your *app* via Facebook Login. Different namespace — a PSID and ASID for the same person do not match and cannot be derived from each other.
- **IGSID (Instagram-Scoped ID):** the analogous ID for Instagram messaging (per professional IG account). Same Send API shape, `object: "instagram"` webhooks.
- **ID Matching API:** for businesses (Pages/apps owned by the same Business Manager) you can map IDs: `GET /<PSID>/ids_for_apps`, `GET /<ASID>/ids_for_pages` (Business Manager linkage required). Useful when merging helpdesk identities across Pages of one business.
- **User Profile API:** `GET /<PSID>?fields=first_name,last_name,profile_pic` with the Page token (fields gated by permissions/consent).

---

## 8. Rate Limits & Policy

### Rate limits (Pages / Messenger, official numbers)

| Scope | Limit |
|---|---|
| Messenger API overall | "Calls within 24 hours = 200 × Number of Engaged Users" (people the business can message). |
| Send API — text, links, reactions, stickers | **300 calls/sec per Page**. |
| Send API — audio/video | **10 calls/sec per Page**. |
| Conversations API | 2 calls/sec per Page. |
| Private Replies | 750 calls/hour per Page. |
| Per-thread | "Your app may be rate limited if too many messages are being sent to a single thread." |

Rate-limit error codes: **80006** (Messenger rate limit reached), **32** (Pages user/app rate limit), **613** (legacy generic rate limit), **error 100** now also returned for deprecated message tags. On 80006/32, back off and retry with jitter; watch the `X-Business-Use-Case-Usage` header.

### Policy essentials for a helpdesk product

1. **User-initiated only** — no cold outreach; respect the 24h window + HUMAN_AGENT 7-day window.
2. **Opt-in for proactive messaging** — marketing sends require explicit, revocable opt-in (MAPI subscription tokens); honor "stop" requests immediately.
3. **Bot disclosure** where required (see §4).
4. **Data use** — message content only for the conversation/service; comply with Platform Terms, data deletion callbacks, and (for tech providers) Business Verification.
5. **Quality signals matter** — block/report rates can get a Page's messaging features restricted; keep response times fast (Meta surfaces responsiveness badges).
6. Webhook reliability is a review requirement: `200 OK` within 20s.

---

## Key Terms

**PSID (Page-Scoped ID)**
: Stable user ID per (user, Page) pair; the only ID you can message.

**ASID (App-Scoped ID)**
: User ID per (user, app) from Facebook Login; not interchangeable with PSID without the ID Matching API.

**IGSID**
: Instagram-scoped user ID for Instagram DMs via the same platform.

**Page Access Token**
: Page-level credential used in `access_token` for all Send API/profile calls; obtain long-lived tokens via OAuth `/me/accounts`.

**Standard messaging window**
: 24 hours after the user's last interaction during which the business may send any message, including promotional.

**Message tag**
: Send API `tag` allowing specific message types outside the window; as of 2026 effectively only `HUMAN_AGENT` (7 days, human replies only, requires `human_agent` permission).

**Utility Template**
: Pre-approved structured template (order/account/appointment updates) — the 2026 replacement for ACCOUNT_UPDATE/POST_PURCHASE_UPDATE/CONFIRMED_EVENT_UPDATE tags.

**Marketing Messages API (MAPI)**
: Opt-in, subscription-token-based proactive marketing channel replacing Recurring Notifications (which died 2026-02-10).

**Handover Protocol / Conversation Routing**
: Mechanism for multiple apps (bot + human inbox) to share one Page's threads via thread-control endpoints; non-owners get `standby` webhooks.

**Primary / secondary receiver**
: The default thread-owning app vs. apps that observe (standby) and can request/receive control. Page Inbox app ID: `263902037430900`.

**Sender action**
: `typing_on` / `typing_off` / `mark_seen` signals sent via the Send API instead of a message.

**Quick reply**
: Up to 13 tappable chips attached to a message; tap returns `quick_reply.payload` in the `messages` webhook.

**Persistent menu / ice breakers / Get Started**
: Messenger Profile API features for always-on navigation, FAQ prompts (max 4), and first-touch CTA.

**m.me link**
: `https://m.me/<PAGE>?ref=...` deep link into the conversation; ref surfaces in `messaging_referrals` / postback referral.

**Webhook watermark**
: Timestamp in `message_deliveries`/`message_reads` meaning "everything before this is delivered/read."

**message_echoes**
: Webhook echoing messages sent *by the Page* from any app — use it to sync agent replies made in Page Inbox into your transcript.

---

## Official Docs

- Platform overview: https://developers.facebook.com/docs/messenger-platform/ (new tree: https://developers.facebook.com/documentation/business-messaging/messenger-platform)
- Get started / app setup: https://developers.facebook.com/docs/messenger-platform/getting-started/quick-start/
- Send API reference: https://developers.facebook.com/docs/messenger-platform/reference/send-api/
- Send messages overview (windows, types): https://developers.facebook.com/docs/messenger-platform/send-messages/
- Quick replies: https://developers.facebook.com/docs/messenger-platform/send-messages/quick-replies/
- Sender actions: https://developers.facebook.com/docs/messenger-platform/send-messages/sender-actions/
- Templates: https://developers.facebook.com/docs/messenger-platform/send-messages/templates
- Webhooks setup: https://developers.facebook.com/documentation/business-messaging/messenger-platform/webhooks
- Webhook event reference (messages): https://developers.facebook.com/docs/messenger-platform/reference/webhook-events/messages/
- Webhook event reference (postbacks): https://developers.facebook.com/docs/messenger-platform/reference/webhook-events/messaging_postbacks/
- Webhook event reference (optins): https://developers.facebook.com/docs/messenger-platform/reference/webhook-events/messaging_optins/
- Conversation routing / handover: https://developers.facebook.com/docs/messenger-platform/conversation-routing/
- Handover reference: https://developers.facebook.com/docs/messenger-platform/reference/handover-protocol
- Standby webhook: https://developers.facebook.com/docs/messenger-platform/reference/webhook-events/standby/
- ID matching (PSID/ASID/IGSID): https://developers.facebook.com/docs/messenger-platform/identity/id-matching
- Persistent menu: https://developers.facebook.com/docs/messenger-platform/send-messages/persistent-menu/
- Ice breakers: https://developers.facebook.com/docs/messenger-platform/reference/messenger-profile-api/ice-breakers/
- m.me links: https://developers.facebook.com/docs/messenger-platform/discovery/m-me-links/
- Policy overview: https://developers.facebook.com/docs/messenger-platform/policy/policy-overview/
- App review: https://developers.facebook.com/docs/messenger-platform/app-review/
- `pages_messaging` permission: https://developers.facebook.com/docs/permissions/reference/pages_messaging
- Rate limiting: https://developers.facebook.com/docs/graph-api/overview/rate-limiting/
- Changelog (tag deprecations): https://developers.facebook.com/docs/messenger-platform/changelog/
- Marketing Messages API FAQ: https://developers.facebook.com/docs/marketing-messages-on-messenger/faq/
- Recurring Notifications sunset notice: https://www.facebook.com/business/help/1321849029608125
