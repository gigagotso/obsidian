# Why operators get signed out when they open many tabs

Investigation of the bug: an operator (user) opens several tabs of the app and, after a
while, gets **automatically signed out** (kicked to `/signin`) across tabs.

There are two distinct failure families. They produce different symptoms — make sure you
know which one you're actually seeing:

| Symptom | Cause family | Where |
|---------|--------------|-------|
| **Redirected to `/signin`, has to log in again** ("signed out") | **OAuth refresh-token rotation race** | Frontend `axios.js` + Passport config |
| **Stays logged in but shows as offline/away on the dashboard** | Presence connection-count leak / sweep race | Backend availability + WS webhooks |

The **primary, most likely cause of an actual sign-out is the refresh-token race** (§1).
The offline-presence issues (§2) are a separate, lower-severity problem that's easy to
mistake for a sign-out. This doc covers both.

---

## 1. PRIMARY CAUSE — OAuth refresh-token rotation race across tabs

### The short version

1. Access tokens live **30 minutes**; refresh tokens are **single-use / rotating** (Passport default).
2. The refresh token is stored in **localStorage, shared by every tab**.
3. Each tab decides to refresh **on its own**, guarded only by a **per-tab in-memory flag**.
4. When the access token expires, **all open tabs try to refresh at the same time, with the same refresh token**.
5. The first tab wins and **rotates (invalidates) that refresh token**; every other tab's refresh now fails with `invalid_grant` (401).
6. A failed refresh **redirects that tab to `/signin` and tears down its WebSocket** → the user is "signed out".

More tabs ⇒ more simultaneous refreshers ⇒ near-certain that all-but-one lose the race.
And it recurs roughly **every 30 minutes** ("at some point"), which matches the report.

### The evidence

**Backend — tokens are short-lived and refresh tokens rotate**

[api/app/Providers/AuthServiceProvider.php:45](../../api/app/Providers/AuthServiceProvider.php):

```php
Passport::tokensExpireIn(now()->addMinutes(30))
        ->refreshTokensExpireIn(now()->addHours(24));
```

Laravel Passport / `league/oauth2-server` **rotate refresh tokens by default**: using a
refresh token issues a new access + new refresh token and **revokes the old refresh
token**. So a given refresh token works exactly once. (Confirmed rotation/revocation usage
in the codebase, e.g. [OAuthController.php:60-61](../../api/app/Http/Controllers/OAuth/OAuthController.php)
revoking access + refresh tokens.) There is **no grace window** that would let a
just-rotated refresh token be reused by a slower tab.

**Frontend — shared token storage, per-tab refresh guard, no cross-tab coordination**

[src/util/axios.js](../src/util/axios.js):

- Token (incl. `refresh_token`) is read from and written to **shared localStorage**
  (`live_caller__oauth`, via Vue2Storage) — lines 32–46, and `refreshAccessToken` writes
  the rotated token back at line 188. All tabs see the same key.
- The de-dupe guard is a **module-scoped boolean**, line 7:

  ```js
  let isRefreshing = false
  ```

  This only prevents *concurrent refreshes within one tab*. Each tab is a separate JS
  context with its own `isRefreshing`, so it does **nothing across tabs**.

- **Proactive refresh** in the request interceptor — every tab independently fires when the
  token is within 60 s of expiry (lines 52–66):

  ```js
  const jwt = jwtDecode(token.access_token)
  if (jwt.exp - 60 < Date.now() / 1000) {
      if (!isRefreshing) {
          isRefreshing = true
          refreshAccessToken(token.refresh_token)   // same refresh_token in every tab
              .then(onRefreshed)
              .catch(onRefreshFailed)
              .finally(() => { isRefreshing = false })
      }
      ...
  }
  ```

  Because the 60-second window and the shared expiry are identical across tabs, **all tabs
  cross the threshold at essentially the same moment** and each sends its own
  `grant_type=refresh_token` request with the same (about-to-be-revoked) refresh token.

- **Reactive refresh / the actual sign-out** in the response interceptor (lines 124–163):

  ```js
  if (status === 401) {
      const token = LiveCaller.app.$storage.get('oauth', {})
      if (token.refresh_token) {
          if (!isRefreshing) {
              isRefreshing = true
              refreshAccessToken(token.refresh_token)
                  .then(onRefreshed)
                  .catch((e) => {
                      onRefreshFailed(e)
                      cleanupWebSocket()                                  // tear down sockets
                      LiveCaller.app.$router.push({ name: 'signin' })     // ← SIGNED OUT
                  })
                  .finally(() => { isRefreshing = false })
          }
          ...
      }
  }
  ```

  When a losing tab's refresh comes back `401 invalid_grant`, this `.catch` is exactly what
  **destroys the WebSocket and redirects to `/signin`**.

- There is **no `storage` event listener and no `BroadcastChannel`/Web Locks** anywhere in
  the app to coordinate refresh or to notice that another tab already wrote a fresh token.
  So a losing tab can't "recover" by picking up the winner's new token — it just signs out.

### The exact race, step by step

```
t0   Access token has ~60s left. Tabs A, B, C all hold refresh_token = R (shared storage).
t1   A's request interceptor: exp-60 < now  → isRefreshing(A)=true → POST /token {refresh_token: R}
t1   B's request interceptor: exp-60 < now  → isRefreshing(B)=true → POST /token {refresh_token: R}
t1   C's request interceptor: exp-60 < now  → isRefreshing(C)=true → POST /token {refresh_token: R}
t2   Server: first to arrive (A) → 200, issues R'. Old R is REVOKED (rotation).
     A writes {access', refresh: R'} to shared localStorage.
t3   B and C arrive with R → 401 invalid_grant (R no longer valid).
t4   Even via the proactive path, B/C then send their queued request with a stale/expired
     access token → 401 → response interceptor 401 handler → refresh with R again → 401 →
     .catch → cleanupWebSocket() + router.push('signin').
==>  Tabs B and C are signed out. Recurs ~every 30 min when the access token next expires.
```

> Subtlety in the proactive path: the request-interceptor promise ends with
> `.finally(() => resolve(config))` (line 72), so even when the refresh *rejects*, the
> original request is still sent — with the **old, now-expired** Authorization header. That
> request then 401s and drops into the reactive handler above, which is the one that
> performs the redirect. Net effect is the same: sign-out.

### Why "many tabs" specifically

- With 1 tab, the single `isRefreshing` guard works fine — one refresh, one rotation, done.
- With N tabs, there is **no shared lock**, so up to N refreshes fire against a single-use
  token. Exactly one can succeed; the other **N-1 are guaranteed to fail and sign out**.
  The probability of a collision goes to ~100% as N grows because all tabs share the same
  expiry deadline and the same 60 s pre-expiry trigger.

### Fix options (frontend, in order of robustness)

1. **Cross-tab single-flight refresh.** Use the **Web Locks API**
   (`navigator.locks.request('token-refresh', …)`) or a `BroadcastChannel`/localStorage
   mutex so only **one tab** ever performs the refresh; other tabs await the result and
   read the rotated token from storage.
2. **`storage` event listener.** When any tab writes a new `oauth` value, other tabs update
   their in-memory token and **retry the pending request instead of signing out**.
3. **Recover instead of redirect on `invalid_grant`.** Before redirecting, re-read
   `oauth` from storage; if the stored `refresh_token` differs from the one we just tried
   (i.e. another tab already rotated it), **retry with the new token** rather than calling
   `router.push('signin')`.
4. **Backend grace window (optional, defense-in-depth).** Allow a just-rotated refresh
   token to be accepted for a few seconds (refresh-token reuse interval) so near-
   simultaneous refreshes don't hard-fail. This is a server change to the OAuth flow.

Option 1 is the proper fix; 2–3 are pragmatic and small.

---

## 2. SECONDARY — operator shows OFFLINE (not actually signed out)

If the symptom is "the operator silently drops to **offline/away** on the dashboard while
their tabs stay logged in", that's a different bug in the presence/availability layer, not
the OAuth flow. Closing one tab can wrongly mark a multi-tab user fully offline.

These all live in the backend and are documented in detail in
[online-users-status.md](./online-users-status.md). Summary of the relevant risks:

### 2a. Presence connection-count leak (`user:` prefix)

[api/app/Broadcasting/OnlineUsersChannel.php](../../api/app/Broadcasting/OnlineUsersChannel.php)
returns `broadcaster_prefix: 'user:'`. The code comment itself documents the hazard: the WS
server (Sockudo) counts connections per `user_id` **string**; if the same socket appears as
`2` on one presence channel and `user:2` on another, cleanup decrements only one key and the
other **leaks**. A leaked count means a later reconnect is treated as "already connected",
`member_added` is skipped, `handleLogin` never runs, and the **DB stays offline**. Many tabs
across `online-users.*` and `conversations.*` channels multiply the chance of a mismatch.

### 2b. Disconnect race in `ProcessSocketEvents`

[api/app/Jobs/ProcessSocketEvents.php](../../api/app/Jobs/ProcessSocketEvents.php) tries to
suppress spurious disconnects with a 30 s "recent login" cache + a `stillPresent()` check.
But `stillPresent()` **returns `false` on any error/timeout** (Sockudo API call), so under
many-tab churn a `member_removed` for one closed tab can be processed as a full
`handleLogout()` even though other tabs are still connected.

### 2c. `SweepAvailability` cron

[api/app/Console/Commands/SweepAvailability.php](../../api/app/Console/Commands/SweepAvailability.php),
scheduled `everyFiveMinutes`, marks users offline when `last_activity_at` is stale. It does
guard with a live presence-member check (`if ($present->has($userId)) continue`), but that
check depends on the same presence membership that §2a can corrupt — so a leaked count can
let the sweep log out a still-connected operator.

### Note on which one you're seeing

`§2` does **not** redirect the browser to `/signin` — the tab stays on the app, just shows
the wrong availability. If users report "I had to log in again", that is `§1`. If they
report "it says I'm offline / I stopped receiving chats but I was still on the page", that
is `§2`.

---

## 3. Recommended next steps

1. **Confirm the symptom.** Reproduce with ~5–10 tabs, watch the Network tab for
   `POST …/token` returning `401 invalid_grant`, and confirm the redirect to `/signin`
   fires from [axios.js:138](../src/util/axios.js). That nails §1.
2. **Quick win:** add the cross-tab refresh lock + `storage`-event recovery (§1 fixes 1–3).
   This is self-contained in [src/util/axios.js](../src/util/axios.js).
3. If instead the report is about availability, prioritise the `user:` prefix
   connection-count fix (§2a) and make `stillPresent()` fail-safe (treat errors as "still
   present" rather than "gone") in [ProcessSocketEvents.php](../../api/app/Jobs/ProcessSocketEvents.php).

---

## 4. File index

**Frontend (`www/app`)**
- [src/util/axios.js](../src/util/axios.js) — token storage, refresh, 401/419 → signin (the §1 bug)
- [src/stores/websocket.js](../src/stores/websocket.js) — `destroy()` called on teardown
- [src/util/Echo.js](../src/util/Echo.js) / [src/util/PusherBatchAuthorizer.js](../src/util/PusherBatchAuthorizer.js) — channel auth
- [src/services/api.services.js](../src/services/api.services.js) — explicit `logOut()`
- [src/store/modules/auth.js](../src/store/modules/auth.js) — `logOut`, IP-whitelist logout
- [src/router/middleware/auth.js](../src/router/middleware/auth.js) — route-guard 401 → signin

**Backend (`www/api`)**
- [app/Providers/AuthServiceProvider.php](../../api/app/Providers/AuthServiceProvider.php) — token TTLs (30 min / 24 h)
- [app/Http/Controllers/OAuth/OAuthController.php](../../api/app/Http/Controllers/OAuth/OAuthController.php) — token revocation
- [app/Broadcasting/OnlineUsersChannel.php](../../api/app/Broadcasting/OnlineUsersChannel.php) — `user:` prefix (§2a)
- [app/Jobs/ProcessSocketEvents.php](../../api/app/Jobs/ProcessSocketEvents.php) — disconnect race (§2b)
- [app/Console/Commands/SweepAvailability.php](../../api/app/Console/Commands/SweepAvailability.php) — sweep cron (§2c)
- [app/Http/Controllers/Webhooks/WsWebhookController.php](../../api/app/Http/Controllers/Webhooks/WsWebhookController.php) — member_added/removed webhook
