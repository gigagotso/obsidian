# Meta Webhooks

**TL;DR:** Webhooks push real-time JSON `POST`s to your HTTPS endpoint when subscribed objects change (Page messages, Instagram DMs, WhatsApp messages…). Setup = (1) expose an endpoint with a valid TLS cert, (2) pass the one-time **GET verification handshake** (echo `hub.challenge` after checking `hub.verify_token`), (3) validate every delivery via **`X-Hub-Signature-256`** (HMAC-SHA256 of the raw body with your app secret), (4) reply `200 OK` fast. Unacknowledged deliveries are retried with decreasing frequency for **up to 36 hours**, then dropped. For Pages you must also subscribe each Page (`/{page-id}/subscribed_apps`); optional **mTLS** hardens the endpoint.

---

## 1. Subscription model

Two layers:

1. **App-level subscription** — which object type + fields your app wants. Configure in App Dashboard (Products → Webhooks, or the product's own panel for Messenger/WhatsApp/Instagram) or via API with an **app access token**:

```http
POST https://graph.facebook.com/v25.0/{app-id}/subscriptions
  ?object=page
  &callback_url=https://example.com/webhooks/meta
  &fields=messages,messaging_postbacks,message_deliveries,message_reads
  &include_values=true
  &verify_token={your-verify-token}
  &access_token={app-id}|{app-secret}
```

2. **Asset-level subscription** — bind each individual asset to your app:
   - Pages/Messenger: `POST /{page-id}/subscribed_apps?subscribed_fields=messages,...` with the **Page token** (needs `pages_manage_metadata`).
   - WhatsApp: `POST /{waba-id}/subscribed_apps` with the business token.
   - Instagram messaging events arrive via the linked Page subscription (or IG Login flavor).

**Objects (topics):** `page`, `instagram`, `whatsapp_business_account`, `user`, `permissions`, `application`, etc. **Fields** are the per-object event kinds (`messages`, `messaging_postbacks`, `feed`, `message_template_status_update`, `account_update`, …).

## 2. Verification handshake (GET)

When you save the callback URL, Meta sends:

```http
GET /webhooks/meta?hub.mode=subscribe&hub.challenge=1158201444&hub.verify_token=my_secret_verify_token
```

Your endpoint must check `hub.verify_token` equals the value you configured, then respond `200` with the **raw `hub.challenge` value** as the body. Anything else fails the setup.

```php
// Note: PHP rewrites '.' to '_' in query param names
if (($_GET['hub_mode'] ?? '') === 'subscribe'
    && hash_equals(config('services.meta.verify_token'), $_GET['hub_verify_token'] ?? '')) {
    echo $_GET['hub_challenge'];
    http_response_code(200);
} else {
    http_response_code(403);
}
```

The verify token is an arbitrary string **you** invent — it only proves the configuration round-trip; real security comes from the signature (§4).

## 3. Event notifications (POST)

Requirements: HTTPS with a valid (not self-signed) certificate. Payload:

```json
{
  "object": "page",
  "entry": [
    {
      "id": "{page-id}",
      "time": 1749550000,
      "messaging": [ { "sender": {"id":"..."}, "recipient": {"id":"..."}, "timestamp": 1749550000123,
                       "message": {"mid":"...","text":"hello"} } ]
    }
  ]
}
```

- Top-level `object` names the topic; `entry` is an **array** — multiple events/assets may be batched in one delivery, and batching is not guaranteed to be stable ⇒ **deduplicate** (e.g., by `mid`/event id).
- For generic objects, each entry carries `changes: [{field, value}]` or just `changed_fields` depending on the dashboard "Include Values" setting; Messenger uses `messaging`, WhatsApp nests under `changes[].value`.
- **Respond `200 OK` immediately** (within seconds); enqueue processing async. Slow responses are treated as failures.
- Ordering is not guaranteed — use timestamps.

## 4. Payload signature: `X-Hub-Signature-256`

Every POST includes `X-Hub-Signature-256: sha256={hex}` — HMAC-SHA256 of the **raw request body** keyed with your **App Secret**. Validate before trusting anything:

```php
$raw = file_get_contents('php://input');           // RAW body — not re-serialized JSON
$expected = 'sha256=' . hash_hmac('sha256', $raw, $appSecret);
if (!hash_equals($expected, $_SERVER['HTTP_X_HUB_SIGNATURE_256'] ?? '')) {
    http_response_code(401); exit;
}
```

```js
const crypto = require('crypto');
const expected = 'sha256=' + crypto.createHmac('sha256', APP_SECRET)
                                   .update(rawBodyBuffer).digest('hex');
const valid = crypto.timingSafeEqual(Buffer.from(expected),
                                     Buffer.from(req.get('X-Hub-Signature-256') || ''));
```

Caveats: compute over the byte-exact body (Meta escapes some characters, e.g. unicode as `\uXXXX` and forward slashes — any re-parse/re-stringify breaks the hash); use constant-time comparison; the legacy `X-Hub-Signature` (SHA-1) header still arrives but use the SHA-256 one.

## 5. Retry behavior

- A delivery is "failed" if your endpoint doesn't return `200` quickly.
- Meta **retries immediately, then with decreasing frequency over the next 36 hours**, after which the event is dropped (Messenger/WhatsApp products may offer limited re-delivery tooling, but treat 36 h as the contract).
- Repeated failure can get the app's subscription disabled with developer alerts.
- Consequences for design: idempotent handlers (retries = duplicates), persistent queue between receipt and processing, and a reconciliation job (poll conversations API) to heal gaps > 36 h.

## 6. mTLS (optional hardening)

Meta can present a client certificate when calling your endpoint:

- Enable client-cert verification on your TLS terminator (nginx/ALB/CDN), trusting the **DigiCert CA chain** Meta publishes, and validate the client cert **CN = `client.webhooks.fbclientcerts.com`**.
- Complements (doesn't replace) signature validation.

## 7. Operational checklist

- [ ] Endpoint returns challenge on GET, `200` instantly on POST
- [ ] Signature check on raw body, constant-time compare
- [ ] App-level subscription fields match what you process
- [ ] Per-asset subscription created at customer onboarding (`/{page-id}/subscribed_apps`, `/{waba-id}/subscribed_apps`)
- [ ] Dedup + idempotency keys; async processing queue
- [ ] Alerting on webhook silence per asset (detect dead subscriptions)
- [ ] `verify_token` and App Secret in secrets storage, rotated procedures documented

---

## Key Terms

- **Topic / object**: The object type subscribed to (`page`, `whatsapp_business_account`, `instagram`, `user`, …).
- **Field**: Per-object event category (`messages`, `feed`, `account_update`, …).
- **Verify token**: Arbitrary string you configure; echoed in the GET handshake to confirm endpoint ownership.
- **`hub.challenge`**: Random integer your endpoint must echo to pass verification.
- **`X-Hub-Signature-256`**: `sha256=` + HMAC-SHA256(raw body, app secret) — payload authenticity proof.
- **`/{app-id}/subscriptions`**: App-level subscription management endpoint (app token).
- **`/{page-id}/subscribed_apps`**: Page-level binding of your app to a specific Page (Page token).
- **mTLS**: Optional mutual TLS; Meta's client cert CN is `client.webhooks.fbclientcerts.com`.

## Official Docs

- Webhooks getting started: https://developers.facebook.com/docs/graph-api/webhooks/getting-started
- Webhooks reference (topics/fields): https://developers.facebook.com/docs/graph-api/webhooks/reference
- App subscriptions endpoint: https://developers.facebook.com/docs/graph-api/reference/app/subscriptions
- Messenger webhooks: https://developers.facebook.com/docs/messenger-platform/webhooks
- WhatsApp webhooks: https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks
- Instagram webhooks: https://developers.facebook.com/docs/instagram-platform/webhooks
