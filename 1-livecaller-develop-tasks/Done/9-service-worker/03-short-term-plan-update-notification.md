# Short-term plan: "new version available" prompt + always-fresh refresh

## Goal (the two requirements)

1. **Notify:** when a new version of the app is deployed, a notification pops up in already-open
   tabs asking the user to refresh.
2. **Fresh on refresh:** any refresh — whether from the prompt button or a manual F5 — must load
   the new version. No cache clearing, no closing all tabs, ever.

Both are achievable **without changing dependencies** and while keeping the desktop-PWA install
working. Requirement 2 is the structural fix (network-first HTML); requirement 1 is a small UI
layer on top of the already-firing `updated()` hook.

## Change 1 — `vue.config.js`: stop precaching `index.html`, serve navigations network-first

This single change satisfies requirement 2: the HTML entry point always comes from the network
(falling back to cache only when offline), so a plain refresh after a deploy gets the new
`index.html` → new hashed bundles → new app. Hashed JS/CSS stay precached (instant loads +
offline keep working).

```js
// vue.config.js  (Workbox 4 syntax — see 02-tech-stack.md for why it differs from the research doc)
pwa: {
    name: 'LiveCaller',
    themeColor: '#ffffff',
    msTileColor: '#ffffff',
    appleMobileWebAppCapable: 'yes',
    appleMobileWebAppStatusBarStyle: 'white',
    workboxPluginMode: 'GenerateSW',
    workboxOptions: {
        // Activate immediately; the changeover race is absorbed by the 5s reload
        // retry — see "Decision: skipWaiting: true" below
        skipWaiting: true,
        clientsClaim: true,
        // Default excludes must be repeated — setting `exclude` replaces them
        exclude: [/\.map$/, /^manifest.*\.js(?:on)?$/, /index\.html$/],
        runtimeCaching: [
            {
                // Workbox 4: matchCallback receives ({ url, event })
                urlPattern: ({ event }) => event.request.mode === 'navigate',
                handler: 'NetworkFirst',
                options: {
                    cacheName: 'html-cache',
                    networkTimeoutSeconds: 4, // fall back to cached HTML if network stalls (offline)
                    expiration: { maxEntries: 4 },
                },
            },
        ],
    },
},
```

### Decision: `skipWaiting: true` (final — after a round-trip through `false`)

We briefly flipped to `false` (2026-06-11) to structurally eliminate the orphaned-reload race
(Gotcha #2). Reverted the same day. What the `false` experiment taught us:

- The race was already absorbed by the proven 5s reload retry — `false`'s benefit was
  downgrading a rare, mitigated 5s pause to nothing.
- Its costs kept surfacing: pre-fix users stranded until a full browser restart;
  `register-service-worker` going deaf after load (Gotcha #3 — a lingering waiting worker hits
  the library's early-return on nearly every load); workers parked in `waiting` indefinitely;
  and after each deploy all hashed assets served via network passthrough (old generation's
  precache stays active) until tabs close — permanently slower repeat loads.

With `true`: existing stuck users self-heal on the next update check (no breaking migration,
no cache clearing), exactly one worker generation is active, the precache always matches the
deployed build, and the changeover race costs at most one 5-second auto-recovered pause per
deploy per tab. All three robustness pieces are kept — retry, staleness guard, and our own
`updatefound` listener — since each still covers a (smaller) real window under `true`.

## Change 2 — update notification UI

The detection already works: `updated()` in `src/registerServiceWorker.js` fires in every open
tab when a new SW is installed (i.e., on every deploy). It currently only `console.log`s.

New module, following the existing composable pattern (mirrors `src/shared/toast/`):

```
src/shared/app-update/
├── useAppUpdate.js      # module-level ref shared by all importers
└── AppUpdateBanner.vue  # persistent banner with a "Refresh" button
```

```js
// src/shared/app-update/useAppUpdate.js
import { ref } from 'vue'

const updateAvailable = ref(false)

export function notifyUpdateAvailable() {
    updateAvailable.value = true
}

export default function useAppUpdate() {
    const reload = () => window.location.reload()
    return { updateAvailable, reload }
}
```

```js
// src/registerServiceWorker.js — replace the console.log in updated()
import { notifyUpdateAvailable } from '@/shared/app-update/useAppUpdate'
// ...
updated() {
    notifyUpdateAvailable()
},
```

`AppUpdateBanner.vue` renders when `updateAvailable` is true — fixed-position, **does not
auto-dismiss** (unlike toasts), text like *"A new version of LiveCaller is available"* with a
**Refresh** button calling `reload()`. Mounted once next to `ToastContainer` in the root layout.
Because of Change 1, the reload is guaranteed to fetch the new version — no SW messaging
(`SKIP_WAITING` postMessage, `InjectManifest`, custom worker) is needed at all.

If the user ignores the banner, nothing breaks: their next manual refresh or next visit loads
the new version anyway (requirement 2 covers them).

**Gotcha discovered in testing — spurious banner after refresh.** "A new SW installed" is not
the same as "this page is outdated": the install runs in the background (downloading all
precached assets) and can finish *after* a reload that already delivered the new build —
`updated()` then fires on a page that is already current (the library also re-emits `updated`
on load whenever a worker sits in `waiting`). Guard implemented in `useAppUpdate.js`: before
showing the banner, fetch `/` with `cache: 'no-store'` and check whether the server's
`index.html` still references the `app.<hash>.js` bundle this page is running. Same bundle →
suppress. The hashed app bundle name is a fingerprint of the whole build (the webpack runtime
inside it changes when any chunk changes), and the check fails open (errors → show the banner).

**Gotcha #2 — first reload after an update hangs (Chromium race, root cause CONFIRMED via
chrome://net-export).** Observed on app.test.livecaller.io: the first reload (banner button *or*
toolbar refresh) right after an update installs+activates sits Pending forever; a second reload
always works instantly. The net-export trace proved what happens: the worker handling the
navigation issues its `NetworkFirst` fetch, and the worker is then terminated by the update
changeover — Chrome **cancels the worker's network fetch at ~1ms** (it never reaches the server)
but **never resolves or re-dispatches the navigation** waiting on that worker's `respondWith`.
The document request is orphaned forever. Explicitly ruled out by the trace: server (responds in
13–16ms when actually asked), connections, and HTTP-cache locking (zero contention on retry).
Mitigation in `useAppUpdate.js`: the banner's `reload()` sets a 5s `setTimeout` that reloads
again — if the first navigation succeeded the page is gone and the timer with it; if orphaned,
the retry cancels it and completes in milliseconds (the trace shows exactly this: cancel at
+1ms, retry at +5,010ms, response at +5,026ms). A manual F5 inside the window can still hang
once — unavoidable from page code. **Resolution: the 5s reload retry in `useAppUpdate.js` is
the accepted mitigation** (we trialled `skipWaiting: false` to remove the race structurally,
but reverted — see the Decision section above).

**Gotcha #3 — with `skipWaiting: false`, `register-service-worker` goes deaf after load.** The
library only subscribes to future installs (`onupdatefound`) when **no** worker was waiting at
page load; if one was waiting it emits a single `updated` and returns. Under `skipWaiting:
false` a waiting worker lingers all session, so that branch is hit on nearly every load and
deploys made *after* page load installed silently — no banner. Fix in
`registerServiceWorker.js` `registered()`: add our own `updatefound`/`statechange` listener
calling `notifyUpdateAvailable()` on `installed`-with-controller. Occasionally double-fires
alongside the library's own emit (when nothing was waiting at load) — harmless, the staleness
guard is idempotent.

## Change 3 — HTTP headers (server/CDN, not repo)

For updates to be *detected promptly*, the browser must always revalidate these two files:

| Path | Header |
|---|---|
| `/index.html` (and `/`) | `Cache-Control: no-cache` |
| `/service-worker.js` | `Cache-Control: no-cache` |
| `/js/*`, `/css/*` (hashed) | `Cache-Control: public, max-age=31536000, immutable` |

## Known edge case: `ChunkLoadError` during the deploy window

After the new SW activates, an old page that lazy-loads a route chunk may request an old-hash
file that the new deploy deleted from the server. Two mitigations (either is fine short-term):

- Keep the previous release's `dist/js`/`dist/css` files on the server for one release cycle, and/or
- Add a `router.onError` handler that reloads once on chunk-load failure (guard with a
  sessionStorage flag to prevent loops).

## Rollout & verification checklist

1. Ship Changes 1+2 in one release. (Header config in parallel.)
2. **Deploy N (the fix itself):** existing users are still served stale HTML by the *old* SW one
   last time; the new SW installs, `skipWaiting`s, and cleans the precache. The banner may not
   appear this once — that's expected.
3. **Deploy N+1 (first real test):** open tabs should show the banner; clicking Refresh (or a
   manual F5) must load the new version. Verify in DevTools → Application → Service Workers
   (no SW stuck in "waiting") and Cache Storage (no `index.html` in the precache, an
   `html-cache` entry exists).
4. **Regression test for the "version goes backwards" case** (see [[01-current-state]]): hard
   reload (`Cmd/Ctrl+Shift+R`), then do a *normal* refresh — the app must stay on the new
   version instead of reverting to the previous one.
5. Verify the desktop "Install app" option still appears (SW + manifest intact).
6. Verify offline: load app, go offline, refresh → cached HTML + assets still serve.

## Explicitly out of scope (later tasks)

- Upgrading `@vue/cli-plugin-pwa` / Workbox 4 → current Workbox.
- `InjectManifest` / custom SW with `SKIP_WAITING` messaging (only needed if we later want
  "activate exactly when the user clicks" semantics).
- A version-polling fallback (e.g. fetching `version.json` on an interval). **Superseded:** we
  instead added periodic `registration.update()` checks (hourly + on tab-visibility) in
  `registerServiceWorker.js` — same effect, no extra build artifacts. Each check is one
  conditional fetch of `service-worker.js` (~304 response when nothing changed); a no-op unless
  a deploy actually happened.
- Instant push-based detection (broadcasting a `new-version` event over Echo from the deploy
  pipeline, clients call `registration.update()` on receipt) — nicest long-term option, needs
  deploy-pipeline integration.

See also: [[00-terms-and-browser-support]] (glossary), [[01-current-state]], [[02-tech-stack]].
