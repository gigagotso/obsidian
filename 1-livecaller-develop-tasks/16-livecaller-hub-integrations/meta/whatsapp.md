# WhatsApp Business Platform — Engineering Knowledge Base

> Audience: engineers building a multi-channel customer messaging product (Livecaller hub).
> Last verified against official docs / current sources: **June 2026**.
> Where the official docs were ambiguous or only third-party sources were available, it is flagged inline.

---

## TL;DR

- Build on the **Cloud API** (Meta-hosted). The **On-Premises API is dead** — its final supported version expired **October 23, 2025**; numbers still registered on it can no longer send or receive messages.
- Hierarchy: **Meta Business Portfolio → WhatsApp Business Account (WABA) → business phone number(s)**. You message via the **Phone Number ID** (a Graph API ID, not the E.164 number).
- Two onboarding paths: register your own number directly via the developer console, or — if you onboard *other* businesses (you almost certainly do) — become a **Tech Provider** and use **Embedded Signup**, which yields a **business integration system user access token** scoped to the client's WABA.
- Messaging rules: **free-form messages only inside the 24-hour customer service window** (opened by an inbound user message). Outside the window you must use **pre-approved template messages** (categories: marketing / utility / authentication).
- **Pricing changed on July 1, 2025**: conversation-based pricing (CBP) was retired in favor of **per-message pricing (PMP)**. You pay **per delivered template message**, priced by template category + recipient country. Non-template (service) messages are **free**. Utility templates sent **inside** an open service window are **free**. Free entry point (Click-to-WhatsApp ads / FB Page CTA) gives a **72-hour fully free window**.
- Business-initiated reach is capped by **messaging limit tiers**: 250 (unverified) → 1K → 10K → 100K → unlimited unique users per rolling 24h, scaling automatically with volume + **quality rating**.
- Everything async comes via **webhooks** on the `whatsapp_business_account` object: inbound `messages` and outbound `statuses` (`sent` → `delivered` → `read`, or `failed`). Subscribe your app **per WABA** via `POST /{WABA_ID}/subscribed_apps`.

---

## 1. Product Family

| Product | Who it's for | API? | Notes |
|---|---|---|---|
| **WhatsApp (consumer)** | End users | No | The app your customers use. |
| **WhatsApp Business App** | Micro/small businesses | No | Free phone app; manual chatting, catalog, labels, basic auto-replies. One number, limited multi-device. Not programmable. A number used here must be deleted/migrated before it can be used on the Platform. |
| **WhatsApp Business Platform (Cloud API)** | Medium/large businesses & software providers | Yes | Meta-hosted Graph API. Up to **1,000 msgs/sec** throughput, 99.9% uptime. This is what you integrate. |
| **On-Premises API** | (retired) | — | **Sunset complete.** Final supported client version expired **Oct 23, 2025**. Since v2.53 (Jan 2024) all new features shipped Cloud-only; new number registration was Cloud-only well before sunset. Do not build against it. |

### Who integrates the Platform

- **Direct business** — a business building for its own number(s) with its own Meta app.
- **Tech Provider** (formerly part of the "ISV" bucket) — a software company (like a multi-channel messaging product) that onboards *client* businesses onto WhatsApp through its own Meta app using Embedded Signup. Clients pay Meta directly (their own payment method / credit line) unless you also become a Solution Partner.
- **Solution Partner / BSP (Business Solution Provider)** — Meta-vetted partners (Twilio, Infobip, 360dialog, Vonage, …) who can additionally **bill on behalf of clients** (share their credit line) and resell access. Becoming one requires a partner application process; Tech Provider status is self-serve (app review + business verification).

> For Livecaller: **Tech Provider with Embedded Signup** is the standard path unless you want to take on Meta billing for clients (Solution Partner).

---

## 2. Object Hierarchy

```
Meta Business Portfolio (business ID; "Business Manager")
│   • Business Verification happens here (unlocks 1K+ tiers, display name, OBA)
│   • Payment method / credit line for WhatsApp charges
│
├── Meta App (developer app — yours as Tech Provider, with
│   whatsapp_business_messaging + whatsapp_business_management permissions,
│   webhook callback URL, Embedded Signup configuration)
│
└── WhatsApp Business Account (WABA)  ←── can be in the CLIENT's portfolio,
    │   • Message templates live HERE (per-WABA)        shared with your app
    │   • Webhook subscription per WABA (subscribed_apps)
    │   • Review status / policy enforcement
    │
    ├── Business Phone Number  →  PHONE_NUMBER_ID (Graph API ID)
    │   • Display name (requires approval / policy review)
    │   • Quality rating (green / yellow / red)
    │   • Messaging limit tier (250 / 1K / 10K / 100K / unlimited)
    │   • Two-step verification PIN
    │   • Official Business Account (verified checkmark) status
    │
    └── (more phone numbers …)
```

Key point for data modeling: **templates belong to the WABA**, **quality rating and messaging limits attach to the phone number** (but see §7 — Meta has been moving limit accounting to the portfolio level), and **webhook subscriptions are per WABA, routed to your app**.

---

## 3. Onboarding Paths

### A. Direct (your own number)

1. Create Meta app → add WhatsApp product. You get a **test number** + **temporary access token (~24h)** for the Getting Started flow.
2. Create/select a WABA, add a real phone number (must receive SMS or voice call for OTP verification; must not be active on consumer/Business App).
3. Register the number: `POST /{PHONE_NUMBER_ID}/register` with a 6-digit **two-step verification PIN**.
4. For production, replace the temporary token with a **System User access token** (see §4).

### B. Embedded Signup (you onboard client businesses — Tech Provider)

1. Become a **Tech Provider**: business verification + App Review for `whatsapp_business_messaging` and `whatsapp_business_management` (Advanced Access).
2. Configure **Embedded Signup** in the App Dashboard and embed the JS SDK / "Login with Facebook" button in your product.
3. Client clicks through the popup: logs into Facebook → creates/selects their **Business Portfolio** → creates/selects a **WABA** → adds + OTP-verifies their phone number → grants your app permissions.
4. The flow returns a short-lived **authorization code** to your frontend (via the SDK message event / response).
5. Backend exchanges it: `GET https://graph.facebook.com/v{N}/oauth/access_token?client_id=...&client_secret=...&code=...` → a **business integration system user access token** scoped to *that client's* WABA/assets.
6. `POST /{WABA_ID}/subscribed_apps` (with the client token) so *your app* receives that WABA's webhooks.
7. `POST /{PHONE_NUMBER_ID}/register` with a PIN to enable Cloud API messaging on the number.
8. Store the token per tenant; use it for all messaging/management calls on that client's behalf.

---

## 4. Tokens, Permissions, Webhook Subscription

| Token type | Lifetime | Use |
|---|---|---|
| **Temporary access token** | ~24 h | Getting-started/testing only (generated in App Dashboard). |
| **System User access token** | Long-lived / non-expiring (no-expiry option) | Production access to **your own** portfolio's WABAs. Create a system user in Business Manager, assign WABA + app assets, generate token with the WhatsApp permissions. |
| **Business integration system user access token** | Long-lived | Produced by the Embedded Signup code exchange; scoped to the **client's** granted assets. The per-tenant production token for Tech Providers. |
| **User access token** | Short-lived | Generally avoid for server-side messaging. |

**Required permissions:**
- `whatsapp_business_messaging` — send/receive messages, media, register numbers (Cloud API).
- `whatsapp_business_management` — manage WABAs, templates, phone numbers, subscribed_apps (Business Management API).
- `business_management` — often additionally needed when managing client Business Portfolio assets.

**Webhooks wiring:** one HTTPS callback URL + verify token at the **app** level (verified via `GET` `hub.mode=subscribe`, echo `hub.challenge`); then subscribe **each WABA** with `POST /{WABA_ID}/subscribed_apps`. A WABA can also carry an **override callback URL** if you need per-tenant endpoints. Validate `X-Hub-Signature-256` (HMAC-SHA256 of raw body with your app secret) on every delivery.

---

## 5. Messaging (Cloud API)

Send: `POST https://graph.facebook.com/v{N}/{PHONE_NUMBER_ID}/messages`

```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "+9955XXXXXXXX",
  "type": "text",
  "text": { "preview_url": false, "body": "Hello from Livecaller" }
}
```

**Message types:** `text`, media (`image`, `audio`, `video`, `document`, `sticker` — upload via `POST /{PHONE_NUMBER_ID}/media` or pass a link), `interactive` (`button` — up to 3 reply buttons; `list` — sectioned list of up to 10 rows; also `cta_url` and `flow` for WhatsApp Flows), `location`, `contacts`, `reaction` (emoji on a prior message), and `template`. Mark inbound messages as read / show typing via the messages endpoint (`status: "read"`).

### The 24-hour customer service window

- Opens/refreshes every time the **user sends you a message**.
- Inside the window: any free-form message type, **free of charge** (service messages).
- Outside the window: **only approved template messages** may initiate contact.

### Template messages

- Created per WABA (`POST /{WABA_ID}/message_templates`), reviewed/approved by Meta (usually minutes–48h). Placeholders (`{{1}}`), headers (text/media), buttons (quick reply, URL, phone, copy code, OTP), and language variants per template name.

| Category | Purpose | Examples | Notes |
|---|---|---|---|
| **MARKETING** | Promotions, offers, re-engagement, anything not strictly transactional | Sales, newsletters, win-back | Subject to **template pacing**; most expensive; per-user marketing frequency caps may silently limit delivery |
| **UTILITY** | Transaction/account-specific, user-initiated context | Order confirmations, shipping updates, billing alerts | **Free when delivered inside an open customer service window**; volume-tier discounts |
| **AUTHENTICATION** | One-time passcodes | Login/verification codes | Fixed OTP formats with copy-code/one-tap buttons; volume-tier discounts |

- Meta can **recategorize** templates that don't match their declared category (a mostly-promotional "utility" template gets billed/treated as marketing).
- **Template pacing** (marketing templates): a new or recently-unpaused marketing template is delivered to an initial slice of the audience; remaining sends are **held** while Meta gathers early quality signals (blocks, reports, reads). Good feedback → held messages released; bad feedback → template **paused** and held messages **dropped**, with notification via WhatsApp Manager / email / `message_template_status_update` webhooks.

---

## 6. Pricing (current model — verified June 2026)

**Conversation-based pricing (CBP) is deprecated.** Effective **July 1, 2025**, Meta charges **per delivered template message** (PMP — per-message pricing):

- **Charged:** each **template** message, when (and only when) it is **delivered**. Rate = template **category** × recipient **country calling code** (rate cards per market on the Meta pricing page).
- **Free:**
  - All **non-template (service) messages** — text, media, interactive, etc. — which by definition can only be sent inside an open customer service window.
  - **Utility templates delivered within an open customer service window.**
  - **Everything for 72 hours** inside a **free entry point window**: user messages you from a **Click-to-WhatsApp ad** or a **Facebook Page call-to-action button** *and you reply within 24 hours* — the window opens when your reply is delivered and lasts 72h, covering all message types including marketing templates. (No reply within 24h → no free window; you're back to paid templates.)
- **Volume tiers (also from July 1, 2025):** utility and authentication template rates **decrease with monthly volume** — the more you send in those categories, the lower the per-message rate. Marketing has no volume discount.
- Authentication has special "authentication-international" rates in some markets.

> Practical billing model for a multi-channel inbox: inbound messages and agent replies within 24h cost **nothing**; costs come almost entirely from **business-initiated templates** (campaigns, notifications, OTP). The `pricing` object in status webhooks (§8) tells you per message whether it was billable and under which category — use it as the source of truth for cost attribution. Exact rates change; always pull the current rate card from the official pricing page rather than hardcoding.

---

## 7. Messaging Limits & Quality Rating

**Messaging limit** = max number of **unique users** you can send business-initiated (outside-window) messages to in a **rolling 24-hour period**. Inbound-triggered service conversation replies are **not** limited.

| Tier | Unique users / 24h | How you get it |
|---|---|---|
| Unverified | **250** | Default for new numbers (display name approved, business not verified) |
| Tier 1 | **1,000** | After **Business Verification** (and number/display name approval) |
| Tier 2 | **10,000** | Automatic upgrade |
| Tier 3 | **100,000** | Automatic upgrade |
| Tier 4 | **Unlimited** | Automatic upgrade |

**Scaling rule:** you're upgraded automatically when, over the trailing **7 days**, you've reached roughly **≥50% utilization of your current limit** while your **quality rating is not low**; Meta evaluates eligibility continuously (third-party sources say ~every 6 hours — not confirmed in official docs). Quality drops can freeze or **downgrade** the tier.

> ⚠️ Note: official docs now describe messaging limits as **calculated at the business-portfolio level and shared across all business phone numbers in the portfolio** (a change from the historical per-phone-number model). Verify against the current Messaging Limits doc when implementing quota logic — this materially affects multi-number tenants.

**Quality rating** (per phone number; visible in WhatsApp Manager, surfaced via the `phone_number_quality_update` webhook): **Green / Yellow / Red**, driven by recent user signals — blocks, reports, and engagement. Low quality → status `Flagged`, possible tier downgrade and template pausing. Mitigations: opt-in only sends, expected content, frequency control, easy opt-out.

**Throughput** (distinct from messaging limits): Cloud API defaults to ~80 messages/sec per number, auto-upgradable to up to 1,000 mps for high-tier, stable-quality numbers.

---

## 8. Webhooks

Object: `whatsapp_business_account`. Main field: `messages` (carries both inbound messages and outbound statuses). Other useful fields: `message_template_status_update`, `phone_number_quality_update`, `account_update`. Retries on non-200 with backoff for up to **7 days**; payloads up to 3 MB; **return 200 fast, process async**; dedupe by message/status ID.

### Inbound text message

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "<WABA_ID>",
    "changes": [{
      "field": "messages",
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {
          "display_phone_number": "15550001111",
          "phone_number_id": "<PHONE_NUMBER_ID>"
        },
        "contacts": [{ "profile": { "name": "Customer Name" }, "wa_id": "995599123456" }],
        "messages": [{
          "from": "995599123456",
          "id": "wamid.HBgM...",
          "timestamp": "1717999999",
          "type": "text",
          "text": { "body": "Hi, I need help with my order" }
        }]
      }
    }]
  }]
}
```

(Media messages carry a media `id` you resolve via `GET /{MEDIA_ID}` → short-lived CDN URL fetched with your bearer token. Interactive replies arrive as `type: "interactive"` with `button_reply` / `list_reply`. User-deleted messages arrive as type `unsupported` with error 131051.)

### Status update (sent → delivered → read | failed)

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "<WABA_ID>",
    "changes": [{
      "field": "messages",
      "value": {
        "messaging_product": "whatsapp",
        "metadata": { "display_phone_number": "15550001111", "phone_number_id": "<PHONE_NUMBER_ID>" },
        "statuses": [{
          "id": "wamid.HBgM...",
          "status": "delivered",
          "timestamp": "1718000050",
          "recipient_id": "995599123456",
          "pricing": {
            "billable": true,
            "pricing_model": "PMP",
            "category": "marketing"
          }
        }]
      }
    }]
  }]
}
```

- `status` values: **`sent`** (accepted/sent by WhatsApp), **`delivered`**, **`read`** (only if recipient has read receipts on), **`failed`**.
- `pricing.pricing_model` is **`PMP`** under the current model (`CBP` and the `conversation` object appear only in legacy/deprecated payloads). `billable: false` for free service/entry-point/in-window-utility deliveries. (Field set evolves — treat unknown keys as additive.)
- **Failed example** — `statuses[].errors`:

```json
"statuses": [{
  "id": "wamid.HBgM...",
  "status": "failed",
  "timestamp": "1718000100",
  "recipient_id": "995599123456",
  "errors": [{
    "code": 131047,
    "title": "Re-engagement message",
    "message": "Re-engagement message",
    "error_data": { "details": "Message failed to send because more than 24 hours have passed since the customer last replied to this number." }
  }]
}]
```

Common error codes to handle: `131047` (outside 24h window — must use template), `131026` (recipient unreachable / can't receive), `131049` / `131050` (per-user marketing limits / user stopped marketing), `131051` (unsupported inbound type), `130472` (user's number in an experiment), `132000`-range (template errors), `131056` (pair rate limit).

---

## 9. Phone Numbers, PIN, Migration

**Requirements for a Platform number:**
- Owned by the business; able to receive the verification **SMS or voice call** (no IVR-blocked lines); short codes not supported.
- Not concurrently active on the consumer app or Business App — delete the account there (or use the official migration flow) first.
- **Display name** must follow Meta's display name policy (relation to business, no generic terms, grammar rules); reviewed/approved — name changes trigger re-review.

**Two-step verification PIN:** every Cloud API registration (`POST /{PHONE_NUMBER_ID}/register`) sets/uses a 6-digit PIN. Keep it stored per number — it's required to re-register and to complete migrations. It can be changed via API when you control the registered number, but it **cannot be disabled via API**.

**Official Business Account (checkmark):** historically a **green** checkmark next to the display name. In **2025** Meta unified verification branding — the green tick became the **blue** Meta Verified-style checkmark across WhatsApp (existing OBAs converted automatically). ⚠️ Third-party sources report the open application path was **suspended around August 2025** pending policy changes (tied to the Meta Verified subscription rollout for businesses); the current application route should be verified in WhatsApp Manager / Meta Verified docs before promising it to clients. OBA status is rare and not required to message — it's cosmetic trust signaling.

**Number migration (between WABAs / BSPs):**
- Migration moves a **phone number** from a source WABA to a destination WABA (this is how clients switch BSPs/Tech Providers, or how you consolidate). Initiated from the **destination** side (create the number in the destination WABA with the migration intent, then verify with OTP).
- Prerequisites: **two-step verification disabled on the source number** (manual, in WhatsApp Manager — not possible via API); business verification completed; source and destination WABAs in good standing (typically same business ID, or BSP-to-BSP with consent); destination payment setup.
- What carries over: display name, **quality rating**, **messaging limit tier**, OBA status. What does **not**: **templates** (WABA-level — recreate and re-approve in the destination WABA) and the source-side webhook wiring (re-do `subscribed_apps`/registration on the destination).
- Downtime is typically seconds-to-minutes around the final OTP/register step if sequenced correctly.

---

## Key Terms

**WABA (WhatsApp Business Account)**
: Container for phone numbers and message templates; the asset a client grants your app access to. Webhook subscriptions attach here.

**Business Portfolio (formerly Business Manager account)**
: Top-level Meta business entity; owns WABAs, apps, payment methods; where Business Verification happens.

**Phone Number ID**
: Graph API ID for a registered business number; the path segment for `/messages`, `/media`, `/register`. Not the same as the E.164 number.

**Cloud API**
: Meta-hosted WhatsApp Business Platform messaging API. The only option (On-Premises retired Oct 2025).

**BSP / Solution Partner**
: Meta-vetted provider that can onboard *and bill* client businesses for WhatsApp usage.

**Tech Provider**
: Developer/ISV that onboards client businesses via Embedded Signup using its own Meta app; clients pay Meta directly.

**Embedded Signup**
: Meta's popup OAuth-style flow embedded in your product; produces a code you exchange for a business integration system user access token scoped to the client's WABA.

**System User token**
: Long-lived server-to-server token tied to a Business Manager system user; the production credential for your own assets.

**Business integration system user access token**
: Long-lived token representing your app's access to a *client's* assets, created automatically by Embedded Signup.

**Customer service window**
: 24h window opened/refreshed by each inbound user message; free-form (and free) messaging allowed inside it.

**Free entry point window**
: 72h fully-free window (all message types) opened when a user contacts you via Click-to-WhatsApp ad or FB Page CTA and you reply within 24h.

**Template message (HSM, legacy term)**
: Pre-approved, parameterized message; the only way to initiate contact outside the service window; categories MARKETING / UTILITY / AUTHENTICATION.

**Template pacing**
: Meta's staged rollout of new/unpaused marketing templates — early sends sampled for quality before the rest are released or dropped.

**PMP (per-message pricing)**
: Current pricing model (since July 1, 2025): pay per delivered template message, by category and recipient country. Replaced CBP (conversation-based pricing).

**Quality rating**
: Green/Yellow/Red signal per phone number from user feedback (blocks, reports); gates messaging-limit tier upgrades.

**Messaging limit**
: Cap on unique users reachable with business-initiated messages per rolling 24h (250 / 1K / 10K / 100K / unlimited).

**Two-step verification PIN**
: 6-digit PIN bound to a registered number; needed for (re)registration; must be disabled manually before migrating a number out.

**OBA (Official Business Account)**
: Verified-checkmark status next to the display name (green → blue after Meta's 2025 verification unification).

**wamid**
: WhatsApp message ID (`wamid.…`) returned on send and echoed in status webhooks — correlate sends to statuses with it.

---

## Official Docs

- Platform overview: https://developers.facebook.com/documentation/business-messaging/whatsapp
- Cloud API get started: https://developers.facebook.com/docs/whatsapp/cloud-api/get-started
- On-Premises sunset notice: https://developers.facebook.com/docs/whatsapp/on-premises/sunset
- Pricing (current, per-message): https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing
- Conversation-based pricing (deprecated, for reference): https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing/conversation-based-pricing/
- Messaging limits: https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits
- Quality / display name policy: https://business.whatsapp.com/policy (Business & Commerce Policies)
- Embedded Signup: https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/overview/
- Tech Providers get started: https://developers.facebook.com/documentation/business-messaging/whatsapp/solution-providers/get-started-for-tech-providers
- Access tokens: https://developers.facebook.com/documentation/business-messaging/whatsapp/access-tokens/
- Send messages reference (`/{PHONE_NUMBER_ID}/messages`): https://developers.facebook.com/docs/whatsapp/cloud-api/reference/messages
- Templates / Business Management API: https://developers.facebook.com/docs/whatsapp/business-management-api/message-templates
- Template pacing: https://developers.facebook.com/docs/whatsapp/business-management-api/message-templates/template-pacing/
- Webhooks setup: https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks/
- `messages` webhook reference: https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/messages/
- Phone numbers: https://developers.facebook.com/documentation/business-messaging/whatsapp/business-phone-numbers/phone-numbers
- Two-step verification: https://developers.facebook.com/documentation/business-messaging/whatsapp/business-phone-numbers/two-step-verification/
- Error codes: https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes
- Business site: https://business.whatsapp.com/products/business-platform

> Note: Meta is mid-migration from `…/docs/whatsapp/…` to `…/documentation/business-messaging/whatsapp/…` URLs; both resolve as of June 2026, but prefer the `documentation/business-messaging` paths going forward.
