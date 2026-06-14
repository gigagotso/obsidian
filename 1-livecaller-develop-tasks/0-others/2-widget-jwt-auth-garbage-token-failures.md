# Widget JWT login failures — explained simply

**Errors in traces:** `jwt_token: Wrong number of segments` (422) on
`POST /v1/widget/auth/login` and `GET /v1/widget/auth/visitor`.

---

## 1. How widget login works (step by step)

There are two "logins" — don't confuse them:

- **Visitor session** — every visitor gets one automatically, even anonymous ones. No JWT needed.
- **User identification (JWT)** — optional. The customer's website tells us "this visitor is John,
  user #123 in my system" by giving us a signed JWT.

The flow:

```
Customer's website                  Our widget (frontend JS)            Our API (backend)
──────────────────                  ────────────────────────            ─────────────────
1. Their server creates a JWT
   (signed with widget secret,
   payload: sub, iat)

2. Their page calls:
   LiveCaller.$emit('user.login',
                    token)  ──────► 3. Widget saves the token into
                                       its config (and sometimes into
                                       localStorage / a cookie, to
                                       survive a page reload)

                                    4. On every widget start-up,
                                       widget sends the token:
                                       - has session? GET /auth/visitor ──► 5. API checks the JWT:
                                       - no session?  POST /auth/login      - no token sent? → fine,
                                                                              anonymous session
                                                                            - valid token? → visitor
                                                                              becomes identified user
                                                                            - garbage token? → 422 error
```

**The critical detail in step 5:** sending **no token at all is OK** (anonymous visitor).
Sending a **broken token is a hard error (422)** — and when that happens, the widget has no
error handling on the login call, so **it gets stuck on loading and never appears**.
That is why "in some cases it does not work".

### 1.1 Real-life network trace — anonymous visitor (no `user.login` at all)

This is the exact request sequence observed in production, with what each step does and
**which code produces it**. Two different tokens are involved — keep them apart:

| token | who creates it | where it travels | purpose |
|---|---|---|---|
| **identification JWT** (`jwt_token` field) | the **customer's server**, signed with `widget.jwt_secret` (HS256) | request **body/query** of the two auth endpoints | "this visitor is user #123 in the customer's system" — optional |
| **session JWT** (`auth.access_token`) | **our API** (tymon/jwt-auth), signed with our own key | `Authorization: Bearer …` **header** of every later request | proves "same visitor/session as before" — always present after first login |

The whole earlier analysis (Issues A–C) is about the *first* token. The trace below shows the
*second* one doing its normal job.

**Request 1 — widget configuration (no visitor involved):**

```
GET https://api-widget.livecaller.io/v1/widget/?widget_id=<uuid>
```

- Sent as soon as the page loads, **before** the widget frame is ever opened — the launcher
  needs theme, position, button texts to render itself.
- Returns everything static about the widget: departments with working-hours time groups
  (`is_open`, `next_open`, timezone), enabled features (`call`, `chat`, `branding`,
  `offline`, `feedback`), pre-chat `fields`, `callback_fields`, theme CSS variables, and the
  full translation dictionary for every language.
- Note the host: `api-widget.livecaller.io` is the **cached/CDN route** — the widget JS
  rewrites `api.` to the cached host for this one call
  (`src/store/modules/widget/index.js:18`). The `?widget_id=` query parameter exists **only
  as a cache key** for the CDN; the API itself identifies the widget by the `X-LC-Widget`
  header (`AuthenticateWidget` middleware), which the axios interceptor attaches to every
  request. No session, no visitor, no JWT of either kind is involved here.

**Request 2 — first-ever visit: anonymous session creation:**

```
POST https://api.livecaller.io/v1/widget/auth/login
body: { "device": { ua, browser, os, platform, engine } }      ← no jwt_token at all
```

- Fired by `createSession()` (`CallerWidget.vue:121`) when the widget frame initializes and
  **no stored session exists** in localStorage.
- Because `jwt_token` is absent, the API's `nullable` rule passes and the controller takes
  the anonymous branch (`VisitorController::store`, line 82): it **creates a brand-new
  Visitor** (auto-named, e.g. "Visitor 6718481", `user_id: null` = not identified) and a
  **Session** row storing the device info + IP.
- The response contains three things:
  - `data.uuid` — the session's uuid;
  - `data.sip` — per-session SIP credentials (server, username, password) used by the
    **call** feature to register a softphone for this visitor;
  - `auth` — the **session JWT**: `access_token` with claims `sub` = visitor uuid,
    `sid` = session uuid, `expires_in: 7776000` seconds = **90 days**
    (`$tokenTTL = 129600` minutes in the controller). You can see it in the pasted token:
    `exp − iat = 7 776 000`.
- The widget persists `auth` into localStorage as `live_caller__session_uuid_<widgetId>`
  (or into cookies when `multi_domain_storage` is on).

**Request 3 — page refresh: session resume ("skipping step 2 somehow"):**

```
GET https://api.livecaller.io/v1/widget/auth/visitor
headers: Authorization: Bearer <stored access_token>           ← this is the "somehow"
```

- On every widget boot, `fetchSession()` (`CallerWidget.vue:87`) checks whether
  `live_caller__session_uuid_<widgetId>` exists in localStorage. It does → it calls the GET
  instead of the POST. There is no magic: the axios **request interceptor**
  (`src/util/axios.js:10-24`) reads that stored token and attaches it as the
  `Authorization: Bearer …` header on every API call. That header is how the backend knows
  who this is.
- Server side (`VisitorController::index`): the `widget` auth guard resolves the visitor
  from the token's `sub` claim, the session from the `sid` claim, and then **refreshes** the
  token (`auth('widget')->setTTL($tokenTTL)->refresh()`) — issuing a *new* access token with
  a fresh 90-day window. The response body is the same shape as the POST (same session uuid,
  same visitor, same SIP credentials) plus the new `auth.access_token`, which the widget
  writes back over the stored one.
- Net effect: **sliding expiration**. As long as the visitor returns at least once every 90
  days, they keep the same visitor identity and conversation history forever, without any
  customer-side JWT. A returning visitor never repeats Request 2.
- If the GET fails — stored token expired/invalid, session deleted, or (Issues A–C) a
  garbage `jwt_token` got appended — the `.catch` falls back to `createSession()`:
  the visitor silently becomes a **new** anonymous visitor (history lost), or, in the
  garbage-token case, the POST 422s too and the widget dies.

**Where the customer's `jwt_token` plugs into this picture:** when the page calls
`user.login`, the identification JWT is simply **added** to the requests above —
as a query parameter on Request 3 (`GET /auth/visitor?jwt_token=eyJ…`) or a body field on
Request 2. The API then attaches the customer's `sub` to the visitor (`user_id` stops being
`null`, name/email/mobile get filled from the JWT payload). Everything else — session JWT,
SIP credentials, 90-day refresh — works identically for anonymous and identified visitors.

---

## 2. Where exactly is the issue?

Three different problems are mixed in the traces. Two are on our side (widget), one is on the
customer's side. **The API backend is NOT broken** — it correctly rejects broken tokens.

> **Before reproducing — what you need to know:**
> - The widget stores its data in localStorage with the prefix `live_caller__`
>   (DevTools → Application → Local Storage): `live_caller__set_user_token`,
>   `live_caller__conversation_<widgetId>`, `live_caller__session_uuid_<widgetId>`.
> - The API identifies the widget by the `X-LC-Widget` header and checks the `Origin` header
>   against the widget's allowed origins — so curl tests need both headers.
> - `POST /auth/login` is throttled to **5 requests per minute** — if reproduction suddenly
>   returns 429, wait a minute.
> - To mint test JWTs with a real widget secret, use tinker inside the Docker container:
>
>   ```bash
>   docker exec -it livecaller-php-api-1 php /var/www/api/current/artisan tinker
>   ```
>   ```php
>   $w = \App\Models\Widget::where('uuid', '<widget-uuid>')->first();
>   \Firebase\JWT\JWT::encode(['sub' => 'repro-1', 'iat' => time()], $w->jwt_secret); // valid
>   ```

### Issue A — customer sends the text `"undefined"` (THE ERRORS YOU PASTED)

- **Whose bug:** the **customer's website**.
- **What happens:** their page runs `LiveCaller.$emit('user.login', token)` but their `token`
  variable is empty/unset, so the widget receives the literal text `"undefined"` instead of a JWT.
  Typical causes: they read the token from localStorage before saving it, or a server template
  printed an empty variable into the page.
- **Our widget's share of the blame:** it does **zero checking**. It takes `"undefined"` and
  ships it to the API on every request (`src/LiveCaller.js:159`). API says 422 → widget dies.

**Detailed mechanics.** The `user.login` handler is just:

```js
// src/LiveCaller.js:159
LC.$on('user.login', (token) => {
    LC.config.merge({ user: { jwt_token: token } })   // whatever arrives goes into config
})
```

and every auth request spreads `config.user` into itself
(`CallerWidget.vue` → `fetchSession()` / `createSession()`), so whatever string sits there
goes on the wire. Important proof point: a **real** JS `undefined` would NOT cause this —
axios silently drops `undefined` values from both JSON bodies and query strings, and the API
accepts a missing `jwt_token` (`nullable`). The traces show the **9-character text**
`"undefined"`, which means the customer's page already converted an unset variable to a
string before handing it to us (e.g. `localStorage.getItem()` of a key that was saved with
`setItem(key, undefined)` — localStorage stores everything as text, so it saved the word
"undefined").

**How to reproduce (2 minutes, any page with the widget):**

1. Open a site with the widget. In DevTools → Application → Local Storage, delete the
   `live_caller__conversation_<widgetId>` key if present (we want *no* active conversation,
   so the widget rerenders immediately instead of parking the token).
2. In the console, simulate the customer's mistake exactly the way it happens in the wild:

   ```js
   localStorage.setItem('my_jwt', undefined);                       // stores the TEXT "undefined"
   LiveCaller.$emit('user.login', localStorage.getItem('my_jwt'));  // passes "undefined" to us
   ```

   (Shortcut with the same result: `LiveCaller.$emit('user.login', 'undefined')`.)
3. Watch the Network tab. The widget rerenders and you'll see:
   - `GET /v1/widget/auth/visitor?jwt_token=undefined` → **422** `Wrong number of segments`
     (only if a previous session existed), then the fallback
   - `POST /v1/widget/auth/login` with body `"jwt_token": "undefined"` → **422** — this is
     byte-for-byte the trace from production.
4. Observe the user-visible damage: the widget stays on its loading state and never appears;
   the console shows an unhandled promise rejection (the missing `.catch` in `createSession()`).

To reproduce the **multi-domain** persistence variant: on a widget with
`multi_domain_storage` enabled and an open conversation, do the same emit — the bad string is
written into the cross-subdomain `set_user_token` cookie and you'll see the failing requests
on the *next* page load too, without emitting again.

You can also hit the API directly, without the widget (proves the backend behaves correctly):

```bash
curl -i -X POST 'https://<api-host>/v1/widget/auth/login' \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -H 'X-LC-Widget: <widget-uuid>' -H 'Origin: https://<allowed-origin>' \
  -d '{"jwt_token":"undefined","device":{"ua":"repro"}}'
# → 422 {"errors":{"jwt_token":["Wrong number of segments"]}}
# Now drop the field entirely:
curl ... -d '{"device":{"ua":"repro"}}'
# → 200, anonymous session — proving "no token" is fine, "garbage token" is fatal
```

### Issue B — widget turns a CORRECT token into `true` (our frontend bug)

- **Whose bug:** **ours, in the widget.** One character.
- **Where:** `src/views/layout/CallerWidget.vue` line 208:

  ```js
  jwt_token: this.$storage.has('set_user_token')   // .has() returns true/false (a boolean!)
  // must be:
  jwt_token: this.$storage.get('set_user_token')   // .get() returns the actual token
  ```

- **When it happens:** customer calls `user.login` with a **valid** token while a chat
  conversation is open → widget parks the token in localStorage → on the next page load it reads
  it back with `.has()` instead of `.get()` → sends `jwt_token: true` → 422 → widget dies.
  This hits customers who did everything **correctly**.

**Detailed mechanics — why `.has()` vs `.get()` gives a different outcome.** They are two
different methods on the widget's storage helper (`src/plugins/storage/storage.js`) answering
two different questions:

- `.has(key)` — *"does this key exist?"* → returns a **boolean** (`true`/`false`), never the content;
- `.get(key)` — *"what is stored under this key?"* → reads and returns the **actual value**.

```js
// localStorage contains:  live_caller__set_user_token → "eyJhbGciOiJIUzI1NiIs…xyz"
this.$storage.has('set_user_token')   // → true                         ("yes, it exists")
this.$storage.get('set_user_token')   // → "eyJhbGciOiJIUzI1NiIs…xyz"   (the token itself)
```

The buggy code uses `.has()` twice — correctly in the `if`, incorrectly as the value
(almost certainly a copy-paste of the line above where `has` was never changed to `get`):

```js
// CallerWidget.vue:207-210
if (this.$storage.has('set_user_token')) {                  // correct: existence check
    LiveCaller.config.merge({
        user: { jwt_token: this.$storage.has('set_user_token') }  // BUG: inside the if,
    })                                                            // this is ALWAYS `true`
    this.$storage.remove('set_user_token')                  // …and the real token is deleted
}
```

So the real JWT is never read, `jwt_token` becomes the boolean `true`, and the next line
destroys the stored token forever. On the wire: the POST keeps it as JSON `true`; the GET
query string stringifies it to `"true"`. Both are one segment, not three → `Wrong number of
segments`. JavaScript can't catch this — both calls are valid and truthy, so it only fails at
runtime, on the API side. (The multi-domain branch ten lines below does the same job
correctly with `cookies.getCookie()` — which is also why multi-domain customers don't see
this pattern.)

**How to reproduce (needs a valid token):**

1. Use a widget with `multi_domain_storage` **off**.
2. Mint a valid JWT for that widget (tinker snippet at the top of this section).
3. Open the site, **start a conversation** in the widget (send any message) — confirm
   `live_caller__conversation_<widgetId>` now exists in localStorage. The open conversation
   is the trigger: it makes `App.vue` park the token instead of applying it immediately.
4. In the console: `LiveCaller.$emit('user.login', '<the valid JWT>')`.
   Confirm `live_caller__set_user_token` appeared in localStorage **containing the real token**
   — proof the storage side is correct and the corruption happens on read-back.
5. **Reload the page** and watch the Network tab:
   - `GET /v1/widget/auth/visitor?jwt_token=true` → **422** `Wrong number of segments`;
   - fallback `POST /v1/widget/auth/login` with body `"jwt_token": true` (JSON boolean) →
     **422** with *two* messages — the Georgian `jwt token უნდა იყოს ტექსტი.` (the `string`
     rule) plus `Wrong number of segments` — exactly the double-message rows in the traces;
   - widget stuck on loading; `live_caller__set_user_token` is gone (the token was consumed
     and destroyed).
6. To prove the one-character fix: change `.has(` to `.get(` on `CallerWidget.vue:208`,
   rebuild, repeat steps 3–5 → the GET now carries the real `jwt_token=eyJ…`, returns 200,
   and the visitor is identified.

### Issue C — token is real but expired / wrong signature

- **Whose bug:** the **customer's server** (wrong signing secret, reused old tokens, or server
  clock slightly off — we allow zero clock tolerance).
- These will always exist. The problem on our side is only that a 422 kills the widget instead
  of falling back to an anonymous session.

**Detailed mechanics.** Validation happens in `app/Http/Rules/ValidJWTSignature.php`, which
calls `firebase/php-jwt` v5.2.0 `JWT::decode()` and forwards the library's exception message
verbatim to the API response. The library checks, in order: token shape (3 segments) →
signature against `widget.jwt_secret` → `iat` not in the future → `exp` not in the past.
Clock leeway is **0 seconds**, so a customer server clock running even 1 second ahead makes
every token invalid (`iat` "in the future") until the clocks drift back — the classic
intermittent failure that "fixes itself". An aggravating widget behavior: the token stays in
`config.user` and is re-sent on **every** widget boot for the rest of the browsing session —
so any pageview later than the token's `exp` re-sends a stale token even though the customer
minted it correctly at login time.

**How to reproduce — mint each bad-token flavor in tinker and emit it:**

```php
// docker exec -it livecaller-php-api-1 php /var/www/api/current/artisan tinker
$w = \App\Models\Widget::where('uuid', '<widget-uuid>')->first();

// 1. expired token:
\Firebase\JWT\JWT::encode(['sub' => 'repro-1', 'iat' => time() - 3600, 'exp' => time() - 1800], $w->jwt_secret);

// 2. clock-skew token (customer's clock "2 minutes ahead"):
\Firebase\JWT\JWT::encode(['sub' => 'repro-1', 'iat' => time() + 120], $w->jwt_secret);

// 3. wrong secret:
\Firebase\JWT\JWT::encode(['sub' => 'repro-1', 'iat' => time()], 'not-the-real-secret');

// 4. the documented-payload surprise — iat only, NO exp:
\Firebase\JWT\JWT::encode(['sub' => 'repro-1', 'iat' => time() - 86400 * 365], $w->jwt_secret);
```

Emit each via `LiveCaller.$emit('user.login', '<token>')` on a page with the widget (no open
conversation) and observe in the Network tab:

| token | API response |
|---|---|
| 1 — expired | 422 `Expired token` |
| 2 — future `iat` | 422 `Cannot handle token prior to <date>` |
| 3 — wrong secret | 422 `Signature verification failed` |
| 4 — one-year-old token, no `exp` | **200, visitor identified** — proof the documented "15-minute lifetime" is not enforced when `exp` is absent |

Token 4 is the documentation-vs-code discrepancy in action: the docs require only `sub` +
`iat` and promise a 15-minute lifetime, but expiry is only ever checked against `exp`.

---

## 3. So: is it backend (API) or frontend (widget)?

| Layer | Verdict |
|---|---|
| **API (backend)** | **Works correctly.** Rejecting garbage is the right behavior. Only optional, small improvements (better error text, logging). |
| **Widget (frontend)** | **This is where our bugs are.** The `.has()`/`.get()` bug, no validation of incoming tokens, and no error handling when login fails (widget gets stuck). |
| **Customer's site** | Source of the `"undefined"` values and the expired/mis-signed tokens. We cannot fix their code, but we must survive it and tell them about it. |

**Short answer: it is a frontend (widget) issue + customer integration issue. The API is fine.**

---

## 4. What WE must implement

### Widget (frontend) — the required fixes

1. **Fix the one-character bug** — `CallerWidget.vue:208`: `.has(` → `.get(`.
   This alone fixes Issue B completely.
2. **Validate tokens before sending** — a token must look like `xxx.yyy.zzz` (3 segments).
   If the customer gives us garbage (`"undefined"`, empty, a boolean):
   - **don't send it as `jwt_token`** (then API happily creates an anonymous session — widget works);
   - **print a clear error in the browser console** so the customer's developer sees it:
     `[LiveCaller] user.login rejected: expected a JWT, received the string "undefined"`;
   - **still report it to our API** in a separate field (e.g. `jwt_error: "string:undefined"`),
     so we keep visibility — see point 4 below.
   Apply this check at all 3 places a token enters the widget: the `user.login` handler
   (`LiveCaller.js:159`), and both read-back spots in `CallerWidget.vue` (localStorage + cookie).
3. **Add error handling to the login call** — `createSession()` in `CallerWidget.vue` has no
   `.catch`. Add one, so even when a 422 does happen (Issue C will always exist), the widget
   falls back to an anonymous session instead of being stuck on loading forever.

### API (backend) — optional but recommended

4. **Log the `jwt_error` field** the widget now sends (widget id, account id, reason).
   Why this matters: if the widget just silently drops garbage, we **lose the only signal**
   that tells us which customers are misintegrated (today the 422 traces are that signal).
   With this log we can alert/dashboard per account and support can contact them proactively.
5. **Better 422 message** for garbage that still arrives raw (old widget builds, bots):
   replace the cryptic library text `"Wrong number of segments"` with something actionable like
   `"jwt_token is not a JWT (received the string 'undefined') — your page passes an unset variable to user.login"`.
6. **Policy decisions (separate tickets):**
   - Docs promise a 15-minute token lifetime, but the code only enforces expiry if the customer
     sets an `exp` claim — a token with only `sub`+`iat` (the documented example!) never expires.
     Either enforce it or fix the docs.
   - `user.update` is documented but the widget never implemented a handler for it — emits are
     silently ignored. Implement or remove from docs.
   - Consider a few seconds of JWT clock leeway (reduces Issue C from customer clock skew).

---

## 5. What the CUSTOMER must implement (their side)

Only relevant for Issues A and C — and we should send them this:

1. **Only call `user.login` when you actually have a token:**

   ```js
   if (token && token.split('.').length === 3) {
       LiveCaller.$emit('user.login', token);
   }
   ```

2. **Generate the JWT fresh on the server for every page load** — never cache it in
   localStorage / never reuse old tokens.
3. **Sign with the correct widget secret** (from the LiveCaller dashboard), algorithm HS256,
   payload at minimum `sub` (their user id) and `iat` (current UNIX time in seconds).
4. **Keep the server clock synced (NTP)** — a clock even 1 second ahead makes tokens invalid.

How they can see the problem themselves: F12 → Network tab → find `auth/login` →
look at `jwt_token` in the request → it says `"undefined"` instead of `eyJhbGci…`.

---

## 6. Ready-to-send customer messages

### For customers sending `"undefined"` (Issue A)

> Hi! We noticed the LiveCaller widget on your site fails to identify your users. Your page
> calls `LiveCaller.$emit('user.login', token)` while the `token` variable is empty — the
> widget receives the text `"undefined"` instead of a JWT, so login fails (and the chat widget
> may not load for those visitors).
>
> You can verify: open your site → F12 → Network → filter `auth/login` → the request's
> `jwt_token` field shows `"undefined"`.
>
> Fix: call `user.login` only after your token is actually available, e.g.
> `if (token && token.split('.').length === 3) LiveCaller.$emit('user.login', token);`
> The token must be generated on your server fresh for each page load (HS256, payload with
> `sub` and `iat`).

### For customers with expired / wrongly-signed tokens (Issue C)

> Hi! The JWTs your server generates for the LiveCaller widget are rejected
> (`Expired token` / `Signature verification failed`). Please check:
> 1) the token is signed with the exact JWT secret from your widget settings;
> 2) tokens are generated fresh per page load, not cached or reused;
> 3) your server clock is NTP-synced (a clock running ahead makes `iat` invalid).
> You can inspect any token at https://jwt.io.

### ქართული ვერსიები (Georgian versions)

**Issue A — მომხმარებელი აგზავნის `"undefined"`-ს:**

> გამარჯობა! შევნიშნეთ, რომ თქვენს საიტზე LiveCaller-ის ვიჯეტი ვერ ახერხებს მომხმარებლების
> იდენტიფიკაციას. თქვენი გვერდი იძახებს `LiveCaller.$emit('user.login', token)`-ს მაშინ, როცა
> `token` ცვლადი ცარიელია — ვიჯეტი JWT ტოკენის ნაცვლად იღებს  ტექსტს `"undefined"`, რის გამოც
> ავტორიზაცია ვერ ხერხდება (და შესაძლოა ჩატის ვიჯეტი ამ ვიზიტორებთან საერთოდ ვერ ჩაიტვირთოს).
>
> შემოწმება თავადაც შეგიძლიათ: გახსენით თქვენი საიტი → F12 → Network → მოძებნეთ `auth/login` →
> მოთხოვნის `jwt_token` ველში ჩანს `"undefined"` ნაცვლად `eyJhbGci…`-ით დაწყებული ტოკენისა.
>
> გამოსწორება: გამოიძახეთ `user.login` მხოლოდ მაშინ, როცა ტოკენი ნამდვილად გაქვთ, მაგალითად:
> `if (token && token.split('.').length === 3) LiveCaller.$emit('user.login', token);`
> ტოკენი უნდა გენერირდებოდეს თქვენს სერვერზე, ყოველი გვერდის ჩატვირთვისას ახლიდან
> (ალგორითმი HS256, payload-ში მინიმუმ `sub` და `iat`).

**Issue C — ვადაგასული ან არასწორად ხელმოწერილი ტოკენები:**

> გამარჯობა! JWT ტოკენები, რომლებსაც თქვენი სერვერი LiveCaller-ის ვიჯეტისთვის აგენერირებს,
> უარყოფილია (`Expired token` / `Signature verification failed`). გთხოვთ, შეამოწმოთ:
> 1) ტოკენი ხელმოწერილია ზუსტად იმ JWT საიდუმლო გასაღებით (secret), რომელიც თქვენი ვიჯეტის
> პარამეტრებშია მითითებული;
> 2) ტოკენები ყოველი გვერდის ჩატვირთვისას ახლიდან გენერირდება და არ ხდება მათი დაქეშვა ან
> ხელახლა გამოყენება;
> 3) თქვენი სერვერის საათი სინქრონიზებულია NTP-ით (წინ წასული საათის შემთხვევაში `iat`
> არავალიდური ხდება).
> ნებისმიერი ტოკენის შემოწმება შეგიძლიათ https://jwt.io-ზე.

---

## 7. One-sentence summary

The API is fine; the widget (frontend) forwards garbage tokens unchecked, has a `.has()`-vs-`.get()`
bug that corrupts even valid tokens, and dies on any auth error — fix those three things in the
widget, add garbage-token logging on the API so misintegrated customers stay visible, and send the
affected customers the message above so they stop passing `"undefined"` to `user.login`.



