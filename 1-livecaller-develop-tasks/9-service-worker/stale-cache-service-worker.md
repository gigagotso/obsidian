# Why production keeps loading old files after a deploy

## TL;DR

You have a **PWA service worker** (Workbox, `GenerateSW`) that **precaches your `index.html`**.
After a deploy the new service worker installs but sits in the **`waiting`** state and never
takes control, so the old service worker keeps serving the **old cached `index.html`** — which
points at the **old hashed bundles**. Content hashing is working perfectly; the problem is that
the one file that *isn't* hashed (`index.html`, the thing that lists which hashed files to load)
is being served from a stale cache. The only way a user escapes today is clearing cache or closing
every tab — which is exactly the symptom you're seeing.

---

## Primer: how service workers actually work

If "service worker" isn't yet a concrete thing in your head, read this first — the bug is obvious
once the model clicks.

### What a service worker *is*

A **service worker (SW)** is a JavaScript file the browser runs **in the background, separate from
any tab**. It can't touch the page's DOM. Its one superpower: it acts as a **proxy that sits between
your page and the network**.

```
        WITHOUT a service worker
  ┌─────────┐                 ┌─────────┐
  │  Page   │ ──── fetch ────▶│ Network │
  └─────────┘                 └─────────┘

        WITH a service worker
  ┌─────────┐     ┌────────────────┐     ┌─────────┐
  │  Page   │────▶│ Service Worker │────▶│ Network │
  └─────────┘     │  (interceptor) │     └─────────┘
                  │       │        │
                  │       ▼        │
                  │ ┌─────────────┐│
                  │ │Cache Storage││  ← private on-disk file store
                  │ └─────────────┘│
                  └────────────────┘
```

Every request the page makes — HTML, JS, images, API calls — hits the SW first. The SW decides, in
code, whether to answer **from its private cache** or let it go to the **network**. That's why it
enables offline/instant loads, and also why it can serve **stale files forever** — it answers
*before* the network is ever consulted.

That private store is **Cache Storage**, separate from the normal HTTP cache. It's what "clear site
data" wipes.

### The lifecycle (this is where the bug lives)

A SW doesn't just "run" — it moves through a strict, browser-enforced lifecycle:

```
  register ──▶ install ──▶ WAITING ──▶ activate ──▶ controlling
                              ▲
                       ⚠️ the bug lives here
```

1. **Register** — the page tells the browser "here's my `service-worker.js`."
2. **Install** — the new SW runs its install step; **Workbox downloads and caches files here**
   (including, for us, `index.html`).
3. **Waiting** ⚠️ — critical rule: **if an older SW is already controlling an open tab, the new one
   does NOT take over.** It installs, then parks in "waiting." It only activates once **every tab**
   controlled by the old SW is **closed** — a refresh does *not* count. For an app people leave open
   all day, that basically never happens.
4. **Activate** — it finally takes over and cleans up old caches.
5. **Controlling** — now it's the proxy intercepting all requests.

Two settings override step 3:
- **`skipWaiting`** = "don't park — activate immediately."
- **`clientsClaim`** = "and take control of already-open tabs now, without a reload."

Without them, the default is "wait politely, maybe forever." **That waiting is half the bug.**

### How requests flow — Workbox strategies

With Workbox you don't hand-write the interceptor; you declare *strategies*:

| Strategy | Behavior | Good for |
|---|---|---|
| **Precache** | Served from cache; stored at install time, never hits network | App-shell assets |
| **CacheFirst** | Cache, then network only if missing | Fonts, images, hashed assets |
| **NetworkFirst** | Network, fall back to cache if offline | HTML, API responses (want fresh) |
| **StaleWhileRevalidate** | Serve cache now, refresh in background | Slightly-stale-OK content |

Our setup uses **Precache for `index.html`** — answered from Cache Storage, network never asked.
**That's the other half of the bug.**

### Which browsers run them

Service workers are a mature, standardized API supported in **all modern browsers**:

- **Chrome / Edge / Brave / Opera** (Chromium) — full support; these also offer the desktop
  **"Install app"** feature (the reason we can't just delete the SW).
- **Firefox** — full support.
- **Safari (macOS + iOS)** — since 11.1 (2018); historically the most limited implementation, but works.

Two hard requirements: **HTTPS only** (or `localhost`), and the SW is **origin-scoped** (one at `/`
controls the whole site). So this affects essentially *all* users on *every* modern browser — not an
edge case.

### Why "precaches your index.html" is the trap

- JS/CSS get **content-hashed names** (`app.3f8a9c.js`) — new build ⇒ new filename ⇒ safe to cache
  forever.
- `index.html` **cannot** be hashed (the browser always fetches it from `/`), and it's the **table
  of contents** that says *which* hashed files to load.
- So the whole scheme depends on the browser getting a **fresh `index.html`** after each deploy.
  Precaching does the exact opposite — it **freezes `index.html`** in Cache Storage, so the browser
  keeps loading the *old* hashes and never asks for the new bundles sitting on the server.

Combine the two halves — a **stale precached `index.html`** (never asks the network) **+ a new SW
stuck in `waiting`** (never takes over) — and you get "old app until I clear cache." The sections
below break this down in more detail.

---

## The pieces involved

### 1. Filename hashing (working correctly)

`vue.config.js`:

```js
filenameHashing: true,
```

Every build emits content-hashed assets:

```
dist/js/app.3f8a9c12.js
dist/css/app.7b2e1f04.css
dist/js/chunk-vendors.91ad33bc.js
```

When file content changes, the hash changes, so the filename changes. This is the standard
cache-busting strategy and it is **doing its job**. Hashed files can safely be cached "forever"
by the browser/CDN because a new build produces a *new* filename.

### 2. `index.html` — the un-hashed entry point

`index.html` cannot itself be hashed: the browser has to request it by a fixed URL (`/`,
`/index.html`). It is the manifest that says *"load `app.3f8a9c12.js` and `app.7b2e1f04.css`"*.

```
public/index.html  →  built to  dist/index.html
                       (with <script src="/js/app.<hash>.js"> auto-injected)
```

So the entire cache-busting scheme hinges on one thing: **the browser must fetch a fresh
`index.html` after a deploy.** If it gets a stale `index.html`, it loads the old hashes and never
even asks the server for the new bundles. The new files sit on the server, fully deployed,
completely ignored.

### 3. The service worker (the actual culprit)

`vue.config.js`:

```js
pwa: {
  name: 'LiveCaller',
  workboxPluginMode: 'GenerateSW',   // ← Workbox auto-generates a service worker
}
```

`src/main.js`:

```js
import './registerServiceWorker'
```

`src/registerServiceWorker.js`:

```js
import { register } from 'register-service-worker'

if (process.env.NODE_ENV === 'production') {
  register(`${process.env.BASE_URL}service-worker.js`, {
    updated() {
      console.log('New content is available; please refresh.')   // ← only logs!
    },
    // ...
  })
}
```

With `@vue/cli-plugin-pwa` + `GenerateSW`, Workbox generates a `service-worker.js` that
**precaches the build manifest — including `index.html`**. From then on, the page is served by the
service worker out of the Cache Storage, **not** from the network.

---

## What actually happens on each deploy (step by step)

1. **First visit ever.** Browser loads `index.html` from the network, runs the app, registers the
   service worker. Workbox downloads and precaches `index.html` + all current hashed assets into
   Cache Storage. The SW becomes the **active controller** of the page.

2. **You deploy a new build.** Server now has a new `index.html` (pointing at new hashes) and new
   hashed bundles. The old hashed bundles may or may not still exist.

3. **User reopens / refreshes the app.** The request for `index.html` is **intercepted by the still-active old service worker**, which answers it **from its cache** — the *old* `index.html`.
   That old HTML references the *old* hashes, so the old app loads. The network is never consulted
   for the new version.

4. **In the background**, the browser does fetch the new `service-worker.js` (it detects the file
   changed because the precache manifest hash changed) and installs it. But a newly installed SW
   does **not** replace the running one. By spec it enters the **`waiting`** state and only takes
   over once **every** tab/client controlled by the old SW is closed.

5. Your `updated()` handler fires at this point — but it only does `console.log(...)`. Nothing
   calls `skipWaiting()`, nothing activates the new SW, nothing reloads. So the new SW waits
   indefinitely while the user keeps the tab open (and SPA users almost never fully close all tabs).

6. **Result:** old files served until the user does a hard reload / clears site data / closes all
   tabs. Exactly your bug.

### Why "clear cache" fixes it

Clearing site data deletes **Cache Storage** *and* unregisters the service worker. With the SW
gone, the next request for `index.html` goes to the network → fresh HTML → fresh hashes → new app.

---

## Why this is NOT a hashing / CDN / HTTP-cache problem

It's easy to misdiagnose this as a regular HTTP caching issue (wrong `Cache-Control` headers on
`index.html`, CDN holding the HTML, etc.). It's worth ruling that out, but the service worker is
the dominant cause here for two reasons:

- A service worker sits **in front of** the HTTP cache and the network. Even with perfect
  `Cache-Control: no-cache` headers on `index.html`, the request is answered from Cache Storage by
  the SW before HTTP caching rules ever apply.
- "Stuck until I *clear cache*" (not just "until the CDN TTL expires") is the signature of Cache
  Storage / a service worker, not of ordinary browser or CDN HTTP caching.

(That said, once the SW is dealt with, you still want `index.html` and `service-worker.js` served
with `Cache-Control: no-cache` so the *next* update is always detected. The hashed assets under
`/js` and `/css` can keep long-lived immutable caching.)

---

## Constraint: the desktop install needs the service worker

We install LiveCaller as a **desktop PWA**, so removing the service worker entirely is **not** an
option. Chromium browsers (Chrome/Edge) only offer "Install app" / open a standalone window when the
site meets PWA install criteria, and one of those is a **registered service worker**. So every
option below *keeps* a service worker — the goal is to stop it serving a stale `index.html`, not to
delete it.

The intentions a service worker can serve are independent of each other:

1. **Offline / caching** — precache the app shell + assets (this is the default the PWA plugin gave
   us, and the *sole* cause of the stale-cache bug).
2. **Installability** — the registered SW that makes the desktop install possible (**the part we
   actually need**).
3. **Push / background sync** — not used here.
4. **Custom network strategies** — fine-grained "network vs. cache" control.

The fix is to keep #2 while changing or dropping #1.

---

## The fix options (no changes made yet — your call)

All three keep the app installable. They differ in how `index.html` is served and how updates
activate.

### Option 1 — Network-first navigations + auto-update  *(recommended)*

**What changes.** Keep Workbox `GenerateSW`, but:

- Stop precaching `index.html` and serve **navigation requests `NetworkFirst`** — the browser hits
  the network first for HTML and only falls back to the cached copy when offline. Hashed JS/CSS stay
  precached (instant load + offline).
- Set `skipWaiting: true` + `clientsClaim: true` so the new SW (carrying the new precache manifest)
  activates immediately instead of sitting in `waiting`.

```js
// vue.config.js → pwa
workboxOptions: {
  skipWaiting: true,
  clientsClaim: true,
  exclude: [/index\.html$/, /\.map$/],   // don't precache the HTML entry point
  runtimeCaching: [
    {
      urlPattern: ({ request }) => request.mode === 'navigate',
      handler: 'NetworkFirst',
      options: {
        cacheName: 'html',
        networkTimeoutSeconds: 4,         // fall back to cache if the network stalls
        expiration: { maxEntries: 4 },
      },
    },
  ],
}
```

**Update behavior.** The next normal reload after a deploy gets fresh HTML → fresh hashes. No cache
clear, no closing tabs. Optionally add a one-time `controllerchange` reload for hands-off updates.

- ✅ Keeps offline support **and** installability; config-only change; bug gone.
- ⚠️ `skipWaiting` swaps the SW under a running page; a forced auto-reload can interrupt a user
  mid-action (guard with a "reload once" flag to avoid loops).

### Option 2 — Network-first navigations + prompt to refresh

**What changes.** Same NetworkFirst-for-HTML behavior, but **don't** auto-activate. The new SW waits;
in `registerServiceWorker.js`'s `updated(registration)` we show a banner ("New version available —
Reload"). On click we message the waiting SW to `skipWaiting()` and reload on `controllerchange`:

```js
// registerServiceWorker.js
updated(registration) {
  showUpdateBanner(() => {
    registration.waiting?.postMessage({ type: 'SKIP_WAITING' })
  })
},
// + on 'controllerchange': window.location.reload()
```

**Implementation note.** A SW that listens for a `SKIP_WAITING` message isn't something `GenerateSW`
emits by default, so this realistically means switching to **`InjectManifest`** with a small custom
SW that adds the `message` listener. More moving parts than Option 1.

- ✅ Never reloads without consent — safest for users mid-call/mid-form; keeps offline + install.
- ⚠️ Relies on the user clicking; if ignored they stay on old code. More code (custom SW + banner UI).

### Option 3 — Minimal service worker, no precaching

**What changes.** Replace the precaching SW (via `InjectManifest`) with a tiny custom SW that exists
only to keep the app installable — it registers, `skipWaiting`s, claims clients, and has a
pass-through `fetch` handler that never calls `respondWith`, so every request goes straight to the
network:

```js
// src/service-worker.js (InjectManifest)
self.addEventListener('install', () => self.skipWaiting())
self.addEventListener('activate', (e) => e.waitUntil(self.clients.claim()))
self.addEventListener('fetch', () => { /* no respondWith → normal network fetch */ })
```

Keep the `pwa` block so the plugin still generates the manifest + icons used for the install.

**Update behavior.** No app cache at all → stale content is structurally impossible. Every load is
fresh, like a normal website.

- ✅ Simplest model; can't get stuck on old files; still installable.
- ⚠️ **No offline support** and no instant-from-cache loads (every visit fetches bundles from the
  network). Usually fine for a dashboard, but a real tradeoff vs. Options 1/2.

---

## How the industry handles this (research)

The options above aren't bespoke — they mirror established, documented practice:

- **Google / Workbox recommend exactly the Option 1 hybrid.** Precache the static hashed assets
  (JS/CSS) for instant repeat loads + offline, but use **network-first for HTML / navigation**, with
  cache as the offline fallback, so "a service worker update should pick up the new application shell
  ... rather than serving stale precached HTML indefinitely." The common concrete recipe is to
  **exclude `index.html` from the precache manifest and add a NetworkFirst navigation route** — which
  is what Option 1 does.
  ([app shell model](https://developer.chrome.com/docs/workbox/app-shell-model),
  [Quasar/Workbox issue #2743](https://github.com/quasarframework/quasar/issues/2743))

- **`vite-plugin-pwa` (the de-facto modern PWA tooling) ships our options as first-class modes:**
  - `autoUpdate` → forces `skipWaiting: true` + `clientsClaim: true` and auto-reloads tabs ⇒ **Option 1**.
  - `prompt` (their **default**) → SW waits, app shows a "new version — reload?" UI ⇒ **Option 2**.

  Their explicit guidance: **"If your application has forms, use the default `prompt` option,"**
  because `autoUpdate`'s auto-reload "can lose data in any tab where the user is filling in a form,"
  and to **decide the strategy before production** (switching later is painful).
  ([auto-update guide](https://vite-pwa-org.netlify.app/guide/auto-update.html),
  [prompt guide](https://vite-pwa-org.netlify.app/guide/prompt-for-update))

- **The `ChunkLoadError` gotcha is well-documented**, not theoretical: a cached `index.html` that
  references hashed chunks deleted by the next deploy causes "Loading chunk failed." Standard fixes
  are network-first HTML + keeping old chunks around for a release, and/or a reload-on-chunk-error
  handler.
  ([CRA issue #3613](https://github.com/facebook/create-react-app/issues/3613))

- **Recovering stuck users / killing a bad SW**: Chrome's official guidance is a **no-op service
  worker that `skipWaiting()`s and has *no* fetch handler** (requests pass straight to the network).
  This underpins "existing users auto-recover" and the Option 3 minimal SW.
  ([removing buggy service workers](https://developer.chrome.com/docs/workbox/remove-buggy-service-workers/))

### Takeaway for LiveCaller

The network-first-HTML + precache-assets caching is the **correct, industry-standard fix regardless
of which option** — that part isn't really a choice. The remaining decision is the *activation*
style. Because LiveCaller is an agent console with **forms and live calls in progress**, both the
Workbox ecosystem and vite-plugin-pwa steer session-heavy apps toward **`prompt` (Option 2)** to
avoid auto-reloading someone mid-call. So:

- **Option 1 (auto-reload)** — seamless updates, small risk of interrupting an active session.
- **Option 2 (prompt)** — never interrupts; strong fit for a live-call/agent dashboard.

## Common to all three

- **Existing users auto-recover.** They already carry the old precaching SW. Because the new SW uses
  `skipWaiting`/`clientsClaim` (or, in Option 3, simply doesn't precache), they get unstuck on their
  next visit — no manual cache clearing.
- **HTTP headers.** Serve `index.html` and `service-worker.js` with `Cache-Control: no-cache` so
  updates are always detected. Keep long-lived immutable caching on the hashed `/js` and `/css`
  assets.

## Recommendation

**Option 1.** It removes the stale-cache bug, keeps both offline support and the desktop install, and
is a config-only change. Choose **Option 2** instead if you'd rather users never get an unexpected
reload, or **Option 3** if you want the simplest possible setup and don't care about offline.

Tell me which option you want and I'll implement it.
