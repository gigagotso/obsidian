# Service worker — tech stack (verified versions)

## The chain, top to bottom

```
@vue/cli-plugin-pwa 4.5.19          (devDependency "~4.5.0")
        │  build-time: generates service-worker.js + web manifest + icons meta
        ▼
workbox-webpack-plugin 4.3.1        (transitive — pinned "^4.3.1" by the plugin)
        │  GenerateSW mode: emits the worker from the webpack asset graph
        ▼
workbox-build / workbox runtime 4.x (what actually runs in the browser)

register-service-worker 1.7.1       (runtime dependency "^1.7.1")
        └─ tiny Vue-team helper: registers the worker, exposes lifecycle
           callbacks (ready/registered/cached/updatefound/updated/offline/error)
           Used only in src/registerServiceWorker.js
```

So there are **two direct third-party libraries** (`@vue/cli-plugin-pwa`,
`register-service-worker`) and one important transitive one (**Workbox 4.3.1** — Google's SW
toolkit, which generates and powers the worker). No hand-written service worker code exists in
the repo.

## What each layer owns

| Layer | Owns | We control it via |
|---|---|---|
| `@vue/cli-plugin-pwa` | Wiring Workbox into the Vue CLI build, manifest.json, meta tags | `pwa: {}` block in `vue.config.js` |
| Workbox (`GenerateSW`) | Precache manifest, caching strategies, SW lifecycle flags | `pwa.workboxOptions` (currently **unset**) |
| `register-service-worker` | Registering the SW + surfacing lifecycle events to app code | `src/registerServiceWorker.js` callbacks |

## ⚠️ Workbox 4 syntax gotchas (matters for the fix)

The original research doc (`stale-cache-service-worker.md`) used Workbox 5+ syntax. We are on
**Workbox 4.3.1**, verified against `node_modules/workbox-build` schemas. Differences that
matter:

- `runtimeCaching[].urlPattern` may be a string, RegExp, or function. **Correction (verified
  in `workbox-routing@4.3.1` source, `route.match({ url, request, event })`):** the callback
  receives `({ url, request, event })` even in v4 — `request` is asserted present, `event` is
  not, so prefer `({ request }) => request.mode === 'navigate'`. (The v4 *docs* only mention
  `url`/`event`, which misled us initially.)
- Handler strings are PascalCase and validated: `'NetworkFirst'`, `'CacheFirst'`,
  `'StaleWhileRevalidate'`, etc. — confirmed supported in 4.3.1.
- `skipWaiting`, `clientsClaim`, `exclude`, `networkTimeoutSeconds`,
  `expiration: { maxEntries }` — all confirmed supported in 4.3.1.
- Setting `exclude` **replaces** the default list, so the defaults must be re-added:
  `[/\.map$/, /^manifest.*\.js(?:on)?$/]`.
- `expiration` requires an explicit `cacheName` (schema-enforced: `.with('expiration', 'cacheName')`).
- Workbox 4 `GenerateSW` loads the runtime via `importScripts` from Google's CDN by default
  (`importWorkboxFrom: 'cdn'`). Fine to leave for the short term; can be set to `'local'` later
  if we want zero external requests.

**Conclusion:** everything the short-term fix needs exists in Workbox 4 — no dependency
upgrades required. Upgrading `@vue/cli-plugin-pwa`/Workbox is a separate, later task.

## Adjacent in-repo tech the fix will touch

- `src/registerServiceWorker.js` — the `updated(registration)` hook is the trigger point.
- `src/shared/toast/useToast.js` — existing toast system, **auto-dismisses after a timeout and
  has no action buttons**, so the "new version" prompt needs its own small persistent banner
  component instead (same composable pattern).
- `VUE_APP_VERSION` (`vue.config.js:31`) — already available for showing the version number in
  the banner if wanted.

See also: [[00-terms-and-browser-support]] (glossary), [[01-current-state]],
[[03-short-term-plan-update-notification]].
