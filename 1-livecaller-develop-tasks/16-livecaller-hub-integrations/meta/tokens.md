# Meta Access Tokens

**TL;DR:** Every Graph API call carries an access token that encodes *who* is acting (user, Page, app, or system user) and *what* they're allowed to do (granted scopes). For a server-side messaging platform the canonical chain is: OAuth login → **short-lived user token (~1–2 h)** → exchange for **long-lived user token (~60 days)** → fetch **long-lived Page tokens (no expiration date)** from `/{user-id}/accounts`, or better, use **System User tokens** (non-expiring by default) for assets your business owns. Always send `appsecret_proof` from servers and inspect tokens with `/debug_token`.

---

## 1. Token types at a glance

| Token type | Acts as | Short-lived | Long-lived | How obtained |
|---|---|---|---|---|
| **User access token** | A person | ~1–2 hours (web login) | ~60 days (exchange; mobile SDK logins are long-lived by default) | Login flow (FB Login / FB Login for Business) |
| **Page access token** | A Facebook Page | ~1–2 h (derived from short-lived user token) | **No expiration date** when derived from a long-lived user token (still invalidatable) | `GET /{user-id}/accounts` or `GET /{page-id}?fields=access_token` |
| **App access token** | The app itself | — | Doesn't expire (invalidated if app secret is reset) | `app_id|app_secret` or `GET /oauth/access_token` |
| **System user access token** | A machine identity in a Business Portfolio | optional 60-day variant | **Never expires by default** | Business Manager UI or Business Management API |
| **Business integration system user token** | A system user auto-created in the *customer's* business | per FB Login for Business config (60 days or never) | per config | FB Login for Business / Embedded Signup `code` exchange |
| **Client token** | App identifier for native clients | n/a | n/a (not secret, not standalone) | App Dashboard → Settings → Advanced |

Lifetimes are documented as approximate and "may change without notice." Token strings have **no guaranteed length** — store them in flexible-size columns (e.g. `TEXT`), not fixed `VARCHAR`s.

## 2. User access tokens

Required "any time your app reads, modifies, or writes a person's Facebook data on their behalf."

- **Short-lived:** ~1–2 hours; produced by web login flows.
- **Long-lived:** ~60 days; native mobile SDK logins return long-lived tokens by default.

### How user tokens are minted: the OAuth flows

**a) Facebook Login (manual / server flow):**

```http
https://www.facebook.com/v25.0/dialog/oauth
  ?client_id={app-id}
  &redirect_uri={https-redirect-uri}      # must match a URI registered in FB Login settings
  &state={csrf-token}                     # CSRF protection — always verify on callback
  &scope=pages_show_list,pages_messaging
  &response_type=code                     # 'code' (server flow), 'token', or 'code token'
```

Exchange the one-time `code` server-side:

```http
GET https://graph.facebook.com/v25.0/oauth/access_token
  ?client_id={app-id}
  &redirect_uri={same-redirect-uri}
  &client_secret={app-secret}
  &code={code}
```

Response: `{"access_token":"...","token_type":"bearer","expires_in":...}` — a **short-lived** user token (then do the long-lived exchange below).

**b) Facebook Login for Business** — the preferred flow for tech providers. You create a **configuration** in the App Dashboard (requires a Business-type app) bundling the token type, assets, and permissions, then invoke login with `config_id` instead of `scope`. A configuration can return either a **User access token** or a **Business Integration System User access token** (with configured expiry: 60 days or never). Serving businesses you don't own requires Advanced Access.

**c) WhatsApp Embedded Signup** — a popup flow built on FB Login for Business (`FB.login` with `config_id`, `response_type: 'code'`, `override_default_response_type: true`). The session-info `message` event (`WA_EMBEDDED_SIGNUP`) gives you the customer's `waba_id` and `phone_number_id`; the `authResponse` gives a `code` you exchange for a business integration system user token (see §5).

### Exchange short-lived → long-lived (server-side only — needs app secret)

```http
GET https://graph.facebook.com/v25.0/oauth/access_token
  ?grant_type=fb_exchange_token
  &client_id={app-id}
  &client_secret={app-secret}
  &fb_exchange_token={short-lived-user-token}
```

Response:

```json
{ "access_token": "{long-lived-user-token}", "token_type": "bearer", "expires_in": 5183944 }
```

`expires_in` ≈ 5,183,944 s ≈ 60 days. There is no "refresh token" — to extend, have the user re-authenticate (or repeat the exchange while the current token is still valid; the docs' supported path is re-login).

## 3. Page access tokens

Let you act *as a Page* (read/send Messenger messages, manage settings, post). "Unique to each Page, admin, and app combination." Requesting user must have a Page role and the app needs `pages_show_list` (plus task-relevant permissions like `pages_messaging`, `pages_manage_metadata`).

### Get Page tokens from a user token

```http
GET https://graph.facebook.com/v25.0/{app-scoped-user-id}/accounts
  ?access_token={user-access-token}
```

Returns each managed Page with `id`, `name`, `access_token`, and the user's tasks on the Page (`ANALYZE`, `ADVERTISE`, `MODERATE`, `CREATE_CONTENT`, `MANAGE`, `MESSAGING`).

Single Page variant:

```http
GET https://graph.facebook.com/v25.0/{page-id}?fields=access_token&access_token={user-access-token}
```

**Lifetime rule:** Page token lifetime follows the user token used to fetch it.
- short-lived user token → short-lived Page token
- **long-lived user token → Page token with no expiration date** ("Long-lived Page access tokens do not have an expiration date and only expire or are invalidated under certain conditions" — see §8).

This is the standard pattern for Messenger integrations: exchange once, store the long-lived Page token per connected Page.

## 4. App access tokens

Identify the app itself — used for app-level operations: managing webhook subscriptions (`/{app-id}/subscriptions`), calling `/debug_token`, reading app metadata.

Two equivalent forms:

1. **Concatenation (no network call):** `{app-id}|{app-secret}` — valid wherever an app token is accepted.
2. **OAuth endpoint:**

```http
GET https://graph.facebook.com/oauth/access_token
  ?client_id={app-id}
  &client_secret={app-secret}
  &grant_type=client_credentials
```

Rules: **server-to-server only**; "never hard-code app access tokens into client-side code or app binaries." They don't expire but die if you reset the app secret. They cannot read most user data — use user tokens for that.

## 5. System user access tokens

System users are machine identities inside a **Business Portfolio** (Business Manager → Business Settings → Users → System users). Two kinds:

- **Admin system user** — full access to all assets owned by/shared with the portfolio by default; can create system users and assign permissions. Limit: 1.
- **Employee (regular) system user** — only accesses assets explicitly assigned. Limits: 1 at Standard Access, 10 at Advanced Access (Ads Management Standard Access tier).

Why preferred for server integrations:

- Not tied to a human (no password changes, no employee offboarding breaking prod).
- **Tokens never expire by default**; optionally generate a 60-day expiring token (`set_token_expires_in_60_days=true` via the Business Management API) which can be renewed via the refresh flow.
- Enable "programmatic, automated actions … without requiring input from an app user or re-authentication."

How to obtain:

1. Business Settings → System users → **Add** → choose role → assign assets (Pages, WABAs, ad accounts) with specific permissions.
2. **Generate New Token** → pick the app + scopes → copy the token (UI), or
3. Via API: `POST /{system-user-id}/access_tokens` with `business_app`, `scope`, `appsecret_proof` (and optionally `set_token_expires_in_60_days`). The system user and the app must belong to the same business.

### Business *integration* system user tokens (customer-side)

When customers onboard via **Facebook Login for Business** (or WhatsApp **Embedded Signup**) with a configuration whose token type is "System-user access token", Meta auto-creates a system user *in the customer's* business and returns a **business integration system user access token** scoped to the assets the customer granted. Expiration (60 days vs never) is set in the login **configuration**. This is the standard way a SaaS platform gets durable, user-independent access to client WABAs/Pages. The Embedded Signup flow returns an OAuth `code` which you exchange server-side:

```http
GET https://graph.facebook.com/v25.0/oauth/access_token
  ?client_id={app-id}
  &client_secret={app-secret}
  &code={code-from-embedded-signup}
```

(No `redirect_uri` is needed for codes produced by the Embedded Signup message flow; this detail is per WhatsApp docs — verify against current Embedded Signup implementation page.)

## 6. Client tokens

Found in App Dashboard → Settings → Advanced → Security. Embedded in mobile/desktop binaries, hence "not secret." Cannot be used alone — always `{app-id}|{client-token}` — and only for a limited set of app-level endpoints (e.g., some SDK and app events APIs).

## 7. Debugging tokens

### Access Token Debugger (UI)
https://developers.facebook.com/tools/debug/accesstoken/ — paste any token to see app, type, expiry, scopes, validity.

### `/debug_token` endpoint

```http
GET https://graph.facebook.com/v25.0/debug_token
  ?input_token={token-to-inspect}
  &access_token={app-token-or-app-developer-token}
```

`access_token` must be an **app access token** of the app that issued `input_token`, or a developer-of-that-app user token. Response `data` fields:

| Field | Meaning |
|---|---|
| `app_id`, `application` | Issuing app |
| `type` | `USER`, `PAGE`, `APP`, `SYSTEM_USER` |
| `is_valid` | Boolean validity |
| `issued_at`, `expires_at` | Unix timestamps; `expires_at: 0` ⇒ never expires |
| `data_access_expires_at` | When *data access* expires (~90 days of user inactivity for some user data) even if the token itself is valid |
| `scopes`, `granular_scopes` | Granted permissions; granular shows per-asset grants (e.g., which Page IDs) |
| `user_id` / `profile_id` | App-scoped user / Page for page-impersonated tokens |
| `error` | Validation error details |

Operational tip: debug tokens at onboarding time and on a schedule; alert before `expires_at` / `data_access_expires_at`.

## 8. Expiration & invalidation causes

A token can be invalidated long before its nominal expiry. Common causes (error 190 + subcode):

- **User changes/resets password** (subcode 460 commonly).
- **User logs out of Facebook** or session expires (463 expired, 461 invalid).
- **User deauthorizes the app** or revokes individual permissions (also surfaces as code 10 / 200-series on calls).
- **Security checkpoint** — user must log in at facebook.com to clear (subcode 459).
- **Two-factor / session invalidated**, account compromised or disabled.
- **Page token specifics:** the user who granted it loses their Page role, the Page is unpublished, or the user's token-granting session is invalidated.
- **App-side:** app secret reset (kills app tokens and `appsecret_proof`s), app loses permission via App Review / Data Use Checkup, app restricted for policy violations.
- **System user tokens:** revoked manually, system user deleted, app removed from the business, or business loses access to the asset.
- **90-day data re-authorization:** for some user data, `data_access_expires_at` forces a fresh login even with a valid token.

Handle 190 by triggering re-authentication; don't retry blindly.

## 9. `appsecret_proof`

Proves a server-side call comes from a party knowing the app secret — protects stolen tokens from being replayed by third parties.

> "The app secret proof is a sha256 hash of your access token, using your app secret as the key." (i.e., HMAC-SHA256(key = app_secret, msg = access_token))

```php
$appsecret_proof = hash_hmac('sha256', $access_token, $app_secret);
```

```bash
curl -G https://graph.facebook.com/v25.0/me \
  -d "access_token=$TOKEN" \
  -d "appsecret_proof=$(printf '%s' "$TOKEN" | openssl dgst -sha256 -hmac "$APP_SECRET" | sed 's/^.* //')"
```

Enforce it in **App Dashboard → Settings → Advanced → Require App Secret**: once on, "all client-initiated calls must be proxied through your backend" or they fail. Recommended ON for any server-based messaging platform. (No `appsecret_time` parameter exists in current docs.)

---

## Key Terms

- **Short-lived / long-lived user token**: ~1–2 h vs ~60 days person-scoped token; upgraded via `fb_exchange_token`.
- **Long-lived Page token**: Page-scoped token with no expiration date, derived from a long-lived user token.
- **App access token**: `app_id|app_secret` identity of the app; server-only.
- **System user**: Machine identity in a Business Portfolio; its tokens never expire by default.
- **Business integration system user token**: System-user token minted in a *customer's* business via FB Login for Business / Embedded Signup.
- **Client token**: Non-secret app identifier for native clients.
- **`/debug_token`**: Endpoint (and Debugger UI) for inspecting any token's validity, scopes, expiry.
- **`appsecret_proof`**: HMAC-SHA256(access_token, key=app_secret) parameter proving server origin.
- **`data_access_expires_at`**: Separate 90-day user-data-access clock independent of token expiry.

## Official Docs

- Access Token Guide: https://developers.facebook.com/docs/facebook-login/guides/access-tokens
- Long-lived tokens: https://developers.facebook.com/docs/facebook-login/guides/access-tokens/get-long-lived
- `/debug_token` reference: https://developers.facebook.com/docs/graph-api/reference/debug_token/
- Access Token Debugger: https://developers.facebook.com/tools/debug/accesstoken/
- Securing requests (`appsecret_proof`): https://developers.facebook.com/docs/graph-api/securing-requests
- System users: https://developers.facebook.com/docs/marketing-api/system-users/overview
- System user token generation API: https://developers.facebook.com/docs/business-management-apis/system-users/install-apps-and-generate-tokens/
- WhatsApp access tokens: https://developers.facebook.com/documentation/business-messaging/whatsapp/access-tokens/
- Manual OAuth flow: https://developers.facebook.com/docs/facebook-login/guides/advanced/manual-flow
- Facebook Login for Business: https://developers.facebook.com/docs/facebook-login/facebook-login-for-business
- WhatsApp Embedded Signup: https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/overview/
