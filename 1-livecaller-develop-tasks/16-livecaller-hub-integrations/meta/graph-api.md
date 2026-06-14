# Graph API Basics: Versioning, Rate Limits, Errors, Batch

**TL;DR:** The Graph API is versioned (`https://graph.facebook.com/v25.0/...`, latest **v25.0**, released 2026-02-18); each version lives **at least 2 years** after the next one ships, then calls auto-forward to the oldest live version. Rate limiting is two-tier: legacy **platform limits** (per app/user) and **Business Use Case (BUC) limits** (per Page/IG account/WABA/ad account) reported in `X-App-Usage` / `X-Business-Use-Case-Usage` headers. Key errors: **190** (bad token), **10/200-series** (permissions), **4/17/32/613/80xxx** (throttling). Up to **50 calls** can be packed into one batch request (each still counts against limits).

---

## 1. Request shape

```http
GET https://graph.facebook.com/v25.0/{node-or-edge}
  ?fields=id,name
  &access_token={token}
  &appsecret_proof={proof}
```

Reads use `GET`, writes `POST`, deletes `DELETE`. `fields=` selects sparse fields; nested expansion is supported (`fields=messages{message,from}`).

## 2. Versioning

- Scheme: `vNN.0` path prefix. Latest: **v25.0** (2026-02-18). Recent: v24.0 (2025-10-08), v23.0 (2025-05-29), v22.0 (2025-01-21), v21.0 (2024-10-02). Cadence is roughly 2–3 releases/year.
- **Guarantee:** each core version is usable for **at least 2 years after the *next* version is released** ("a version will no longer be usable two years after the date that the subsequent version is released"). Some platform-specific versions (e.g., Marketing API historically) have shorter windows — check the changelog per product.
- **After expiry:** calls to a dead version are **auto-upgraded** (redirected) to the oldest still-available version — meaning behavior can change silently. Pin versions explicitly and schedule upgrades.
- **Unversioned calls** default to the version set in App Dashboard → Settings → Advanced (JS SDK requires an explicit version).
- You can't call versions older than your app (apps can only use versions that were available when the app started calling them).
- Meta reserves the right to make out-of-band changes for security/privacy.

## 3. Rate limiting

### 3.1 Platform rate limits (apps using user/app tokens)

- **App-level:** `calls per hour = 200 × number of daily active users` — pooled across the app, so one user may exceed 200/h if others are quiet.
- **User-level:** per-user limits exist across apps; exact values undisclosed "due to privacy concerns."
- Monitored via the **`X-App-Usage`** response header: `{"call_count":%, "total_cputime":%, "total_time":%}` — throttled at 100%.

### 3.2 Business Use Case (BUC) rate limits

Apply to Pages API, Instagram Platform, WhatsApp, Marketing API when using page/system-user tokens. Tracked **per business asset**, reported in **`X-Business-Use-Case-Usage`**: `{"{business-id}": [{"type":"pages","call_count":%,"total_cputime":%,"total_time":%,"estimated_time_to_regain_access":minutes}]}`.

| Use case | Documented formula |
|---|---|
| Pages (page/system token) | calls per 24 h = **4800 × engaged users** |
| Instagram Platform | calls per 24 h = **4800 × impressions** (per IG account) |
| WhatsApp Business Mgmt API | 200 req/h default; 5,000/h for active WABAs (Cloud API *messaging* throughput is separate: pair-rate + per-number MPS tiers) |
| Ads Insights | 600–190,000 base calls/h by tier + per-active-ad factor |

### 3.3 Throttling errors & handling

| Code | Meaning |
|---|---|
| 4 | App-level platform limit |
| 17 | User-level platform limit |
| 32 | Page-level platform limit (page+user pair) |
| 613 | Generic "calls exceeded the rate limit" (legacy/misc throttle) |
| 80001 / 80002–80009 | BUC limits (Pages/system token; Instagram, LeadGen, Catalog, WhatsApp, …) |

When throttled: **stop calling** (continuing extends the penalty window), honor `estimated_time_to_regain_access`, back off exponentially, spread load, request fewer fields, dedupe, and prefer webhooks over polling.

## 4. Error handling

Error envelope:

```json
{
  "error": {
    "message": "Error validating access token: ...",
    "type": "OAuthException",
    "code": 190,
    "error_subcode": 460,
    "error_user_title": "...",
    "error_user_msg": "...",
    "fbtrace_id": "EJplcsCHuLu"
  }
}
```

| Code | Meaning | Action |
|---|---|---|
| **190** | Access token invalid/expired (`OAuthException`) | Re-authenticate. Subcodes: 458 app not installed, 459 checkpoint, 460 password changed, 463 expired, 464 unconfirmed user, 467 invalid token |
| **102** | API session issue | If no subcode: get a new token |
| **10** | Permission denied / not granted / removed | Re-request permission or degrade feature |
| **100** | Invalid parameter / unknown field or node | Fix the request (also covers missing object & type mismatches) |
| **200–299** | Specific permission missing (e.g., 200 generic, 230 needs `pages_messaging`) | Handle missing permission |
| **368** | Temporarily blocked for policy violation | Wait and retry later |
| **4 / 17 / 32 / 613 / 80xxx** | Rate limits (see §3.3) | Back off |
| **1 / 2** | Unknown / service error | Retry with backoff |

Log `fbtrace_id` — Meta support needs it. Treat `(code, error_subcode)` pairs, not message strings (messages change).

## 5. Batch requests

- `POST https://graph.facebook.com` with a `batch` parameter: JSON array of up to **50** operations, each `{method, relative_url, body?, name?, headers?}`.
- **Each sub-request counts toward rate limits individually** — batching saves HTTP round-trips, not quota.

```bash
curl -X POST "https://graph.facebook.com/v25.0" \
  -F "access_token=$TOKEN" \
  -F 'batch=[
    {"method":"GET","relative_url":"me"},
    {"method":"GET","relative_url":"me/accounts"}
  ]'
```

- Response: JSON array in the same order; each item has `code` (HTTP status), `headers`, and `body` (JSON-encoded string — double-decode).
- **Dependencies:** name an op (`"name":"get-pages"`) and reference its result with JSONPath: `"relative_url":"{result=get-pages:$.data.*.id}?fields=name"`. Use `omit_response_on_success=false` on the named op if you also want its body.
- **Partial failure:** one failed op doesn't abort the others; failures return their own error envelopes. On timeout, incomplete ops come back as `null` — retry just those.

---

## Key Terms

- **Node / edge / field**: Object, connection, property — the Graph data model.
- **Version pinning**: Calling `vNN.0` explicitly; unversioned calls use the dashboard default.
- **Auto-upgrade**: Expired-version calls silently forwarded to the oldest live version.
- **Platform rate limit**: Legacy per-app (200 × DAU/h) and per-user limits; `X-App-Usage` header.
- **BUC rate limit**: Per-business-asset limits (Pages, IG, WhatsApp, Ads); `X-Business-Use-Case-Usage` header.
- **`fbtrace_id`**: Per-request trace ID for Meta support.
- **Batch request**: Up to 50 ops in one HTTP call, order-preserved responses, JSONPath chaining.

## Official Docs

- Graph API overview: https://developers.facebook.com/docs/graph-api/overview
- Versioning: https://developers.facebook.com/docs/graph-api/guides/versioning
- Changelog (versions & dates): https://developers.facebook.com/docs/graph-api/changelog
- Rate limiting: https://developers.facebook.com/docs/graph-api/overview/rate-limiting
- Error handling: https://developers.facebook.com/docs/graph-api/guides/error-handling
- Batch requests: https://developers.facebook.com/docs/graph-api/batch-requests
