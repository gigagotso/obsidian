# Classic (vanilla) service workers vs Workbox — implementation, terms, best practices

Companion to [[00-terms-and-browser-support]] (general glossary). This doc goes one level
deeper: how you'd build a service worker **by hand**, what Workbox replaces in that picture,
each side's vocabulary, and the best practices that apply regardless of tooling.

---

## Part 1 — How a classic service worker is implemented

A vanilla PWA has exactly three pieces: **page code** that registers the worker, **one worker
file** (`sw.js`), and the **Cache Storage** the worker manages. No build step required.

### 1. Registration (page code)

```js
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')   // scope defaults to '/'
}
```

That's all the page does. Everything else lives in the worker file. (Our app wraps this in the
`register-service-worker` helper, which adds the lifecycle callbacks like `updated()`.)

### 2. The worker file — three event handlers

A hand-written `sw.js` is essentially three event listeners around the lifecycle:

```js
const CACHE = 'app-v2'   // ← manual versioning: bump on every release!
const ASSETS = ['/index.html', '/css/app.css', '/js/app.js']  // ← manual list

// INSTALL — runs once per new worker version: download everything into the cache
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE).then((cache) => cache.addAll(ASSETS)),
    )
    // self.skipWaiting()  // optional: don't park in `waiting`
})

// ACTIVATE — the new version took over: delete caches of previous versions
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((keys) => Promise.all(
            keys.filter((k) => k !== CACHE).map((k) => caches.delete(k)),
        )),
    )
    // event.waitUntil(self.clients.claim())  // optional: control open tabs now
})

// FETCH — the proxy: decide cache vs network for every request
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request)
            .then((cached) => cached || fetch(event.request)),  // cache-first
    )
})
```

A hand-written **network-first** (what our HTML route needs) is noticeably more code:

```js
self.addEventListener('fetch', (event) => {
    if (event.request.mode !== 'navigate') return
    event.respondWith(
        fetch(event.request)
            .then((response) => {
                const copy = response.clone()                 // body is single-use!
                caches.open('html').then((c) => c.put(event.request, copy))
                return response
            })
            .catch(() => caches.match(event.request)),        // offline fallback
    )
})
```

### 3. Classic-SW terms & definitions

- **`self` / `ServiceWorkerGlobalScope`** — the worker's global object (there is no `window`,
  no DOM). `self.skipWaiting()`, `self.clients`, `self.addEventListener` live here.
- **`event.waitUntil(promise)`** — "keep me alive until this finishes." Without it the browser
  may kill the worker mid-install/mid-activate and your caching silently half-completes.
- **`event.respondWith(promise<Response>)`** — the interception primitive: "I'll answer this
  request." Not calling it = the request falls through to the network normally. (Our traced
  hang was a navigation waiting on a `respondWith` whose worker had been terminated.)
- **Caches API** — `caches` (the `CacheStorage` registry) → named `Cache` objects →
  `cache.addAll/put/match/delete`. This is Cache Storage; entries never expire on their own.
- **`clients` / `clients.claim()`** — `clients` = the open tabs/windows in scope;
  `claim()` makes the just-activated worker control already-open tabs without a reload.
- **`self.skipWaiting()`** — promote this installed worker to active immediately instead of
  parking in `waiting` until all tabs close.
- **Registration object** — what `register()` resolves to; exposes the version slots
  `registration.installing` / `.waiting` / `.active`, plus `registration.update()` (force an
  update check) and the `updatefound` event.
- **`statechange`** — event on an individual worker as it moves
  `installing → installed → activating → activated` (or `redundant` = discarded).
- **`importScripts(url)`** — synchronous script loading inside a worker (no ES modules in
  older targets); blocks worker boot until fetched — why a CDN import here is dangerous.
- **Cache-name versioning** — the vanilla update mechanism: new release ⇒ new `CACHE` string ⇒
  install repopulates, activate deletes the old. Forget the bump and users keep stale files.
- **`message` / `postMessage`** — the only page↔worker channel; classically used for the
  "SKIP_WAITING" pattern (page tells the waiting worker to activate when the user clicks
  *Reload*).

### 4. Why hand-written workers go wrong (the classic failure modes)

- The **asset list and cache version are maintained by humans** — forget one file or one bump
  and you ship a stale or broken cache. (Hashed filenames make manual lists impossible in
  practice — a new hash every build.)
- **Re-downloading everything** on each version bump — vanilla cache-name versioning can't
  tell which files actually changed.
- Forgetting `waitUntil` → truncated installs; forgetting `clone()` → "body already used"
  errors; caching error responses or opaque cross-origin responses by accident; no cache
  expiration → quota exhaustion.
- Precaching `index.html` cache-first — *our* original sin; nothing in the vanilla API warns
  you.

---

## Part 2 — What Workbox is, and how it differs

**Workbox** is Google's service-worker library suite. It does not change what a service worker
*is* — the same lifecycle, the same Cache Storage — it generates and/or wraps the handlers
above so the failure-prone parts are computed instead of hand-maintained.

The two integration modes:

- **`GenerateSW`** (what we use, via `@vue/cli-plugin-pwa`): you write **zero worker code**;
  the build emits the whole `service-worker.js` from config (`workboxOptions` in
  `vue.config.js`).
- **`InjectManifest`**: you write your own worker file using Workbox modules, and the build
  injects the precache manifest into it. Needed only for custom logic (push, message handlers,
  exotic routing).

### Vanilla → Workbox mapping

| Hand-written piece | Workbox equivalent | What you gain |
|---|---|---|
| `ASSETS` array + `cache.addAll` in `install` | **precache manifest**, generated at build time | every built file included automatically, with a **revision hash** each |
| `CACHE = 'app-v2'` manual bump | per-file revisions in the manifest | only *changed* files re-download on update; no human bump |
| `activate` cache-cleanup loop | automatic outdated-precache cleanup | can't forget it |
| `fetch` listener `if/else` chains | `registerRoute(matchCallback, strategy)` | declarative routes; ordering and method handling solved |
| hand-rolled cache/network logic | `workbox.strategies.*` (`NetworkFirst`, `CacheFirst`, `StaleWhileRevalidate`, `NetworkOnly`, `CacheOnly`) | `clone()` handling, timeouts (`networkTimeoutSeconds`), error fallbacks done correctly |
| `self.skipWaiting()` call | `skipWaiting: true` config | same semantics, one flag |
| `clients.claim()` call | `clientsClaim: true` config | same |
| ad-hoc expiration code | `expiration` plugin (`maxEntries`, `maxAgeSeconds`) | quota-error purging included |
| "don't cache errors" checks | `cacheableResponse` plugin | declarative statuses/headers |
| page↔worker update messaging | `workbox-window` helper library | typed lifecycle events (we use `register-service-worker` instead — predates it) |

### Workbox terms & definitions

- **Precache manifest** — the build-generated array of `{ url, revision }` entries compiled
  into the worker. The revision changes when file content changes; this is what makes the
  browser see "new worker version" after every deploy. (v4 ships it as a separate
  `precache-manifest.<hash>.js` loaded via `importScripts`; v6+ inlines `self.__WB_MANIFEST`.)
- **`precacheAndRoute(manifest)`** — installs the manifest into Cache Storage *and* registers
  the route serving those files cache-first. The single call that replaces the whole vanilla
  install/fetch pair for static assets.
- **`registerRoute(capture, handler, method)`** — declarative routing. `capture` is a string,
  RegExp, or **match callback** receiving `({ url, request, event })` (verified in our
  v4.3.1 — `request` is asserted present; prefer destructuring it over `event`).
- **Strategy** — a class encapsulating one cache/network decision pattern, configurable with
  `cacheName`, `networkTimeoutSeconds`, `fetchOptions`, `matchOptions`, and plugins.
- **Plugin** — an object hooking strategy lifecycle points (`cacheWillUpdate`,
  `cachedResponseWillBeUsed`, …). Built-ins: `expiration`, `cacheableResponse`,
  `broadcastUpdate`, `backgroundSync`.
- **`runtimeCaching`** — the `GenerateSW` config key that compiles to `registerRoute(...)`
  calls; how we declared the `NetworkFirst` HTML route without writing worker code.
- **`navigateFallback`** — serve a precached shell for all navigations (SPA offline pattern).
  **The opposite of our fix** — it pins navigations to a cached `index.html`. We deliberately
  don't use it.
- **Modules** — `workbox-core`, `workbox-routing`, `workbox-strategies`, `workbox-precaching`,
  `workbox-expiration`…; in v4 exposed as the `workbox.*` namespace via `importScripts`
  (from CDN by default — we switched to `importWorkboxFrom: 'local'`), in v5+ as ES modules
  bundled into the worker.
- **`workbox-window`** — page-side companion (registration + update UX events). Not used here;
  `register-service-worker` fills that role.

### What Workbox does NOT solve

Worth stating, since every one of these bit us:

- **What to precache is still your decision** — it happily precaches `index.html` by default,
  which is exactly the stale-cache bug.
- **Update UX is still your decision** — banner vs auto-reload vs nothing.
- **HTTP headers are still your job** — `no-cache` on the worker file and HTML.
- **Browser bugs are still browser bugs** — the orphaned-navigation race happens below
  Workbox.

---

## Part 3 — Best practices (tool-agnostic)

The short list, each traceable to a documented source or to scars from this task:

1. **Never serve the HTML entry point cache-first.** Precache hashed/immutable assets;
   navigations get `NetworkFirst` (offline fallback) or network-only. This single rule
   prevents the entire "old app after deploy" class of bugs.
2. **Headers:** `Cache-Control: no-cache` on `service-worker.js` and `index.html`;
   `max-age=31536000, immutable` on hashed assets. The SW byte-check is your update trigger —
   never let a CDN cache it.
3. **Decide the update-activation policy deliberately, before production**, and pair it with
   matching UX: `skipWaiting` + prompt/reload handling (us), or waiting + `SKIP_WAITING`
   message. Switching policies later is painful (we round-tripped and found two hidden
   dependencies — see [[03-short-term-plan-update-notification]] gotchas).
4. **If you use `skipWaiting`, plan for version skew:** the running page is old while caches
   are new — keep previous-release hashed assets on the server for one cycle and/or add a
   chunk-error reload guard.
5. **Give long-lived tabs an update path:** periodic `registration.update()` + on
   `visibilitychange`. The browser alone only checks on navigations (+24h cache cap).
6. **Don't load the worker's own code from a third-party CDN** — worker boot blocks on
   `importScripts`; self-host (`importWorkboxFrom: 'local'` in v4, automatic in v5+).
7. **Cache only what you mean to cache.** No API responses by default; never authenticated
   data without an explicit expiration + invalidation story. Bound every runtime cache
   (`maxEntries`/`maxAgeSeconds`).
8. **Verify update *flows*, not just first install:** test deploy → open-tab detection →
   banner → reload → no ghost banner; use DevTools → Application and
   `chrome://serviceworker-internals`. First-install testing catches almost nothing.
9. **Know the kill switch:** a no-op worker (`skipWaiting` + no fetch handler) is Chrome's
   documented recovery from a bad worker in production. You can always deploy your way out —
   users never need to clear caches.
10. **Keep the worker's job minimal.** Every feature in the proxy path (the thing that can
    serve your whole app stale) should justify itself. Installability needs only a registered
    worker + manifest; offline and instant-load are opt-ins, not obligations.

See also: [[00-terms-and-browser-support]], [[01-current-state]], [[02-tech-stack]],
[[03-short-term-plan-update-notification]].
