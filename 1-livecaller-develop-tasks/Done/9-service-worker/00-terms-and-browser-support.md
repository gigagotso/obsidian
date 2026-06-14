# Service worker — terms & definitions, browser support

Reference glossary for the other docs in this folder. Skim once; come back when a term in
[[01-current-state]], [[02-tech-stack]] or [[03-short-term-plan-update-notification]] is unclear.
For a deeper dive — how a hand-written worker is built, the full Workbox vocabulary, and
tool-agnostic best practices — see [[04-classic-sw-and-workbox]].

## Terms & definitions

### Core concepts

- **Service worker (SW)** — a JavaScript file the browser runs in the background, separate from
  any tab. It cannot touch the DOM; its one superpower is acting as a **programmable network
  proxy**: every request the page makes (HTML, JS, images, API calls) hits the SW first, and the
  SW decides in code whether to answer from cache or let the request go to the network.
- **Cache Storage** — the SW's private on-disk file store (`caches.*` API). Separate from the
  normal HTTP cache; this is what "Clear site data" wipes. Our stale `index.html` lives here.
- **HTTP cache** — the browser's ordinary cache governed by `Cache-Control` headers. A SW sits
  *in front of* it: if the SW answers from Cache Storage, HTTP caching rules never even apply.
- **Scope** — the URL range a SW controls. A SW served from `/` controls the whole origin. One
  origin, one app-level SW (ours).
- **Registration** — the page telling the browser "here's my `service-worker.js`"
  (`navigator.serviceWorker.register(...)` — wrapped for us by `register-service-worker`).
- **Client** — any open tab/window (or PWA window) under the SW's scope.
- **Controller** — the SW instance currently intercepting a client's requests. A page can have
  a *registered* SW that is not yet its *controller* (see "waiting").

### Lifecycle

- **Install** — first phase of a new SW version. Workbox downloads and precaches files here.
- **Waiting** — a newly installed SW does **not** replace the currently-controlling one. By
  spec it parks in `waiting` until **every** client of the old SW is closed — a refresh does
  *not* count. This is half of our bug.
- **Activate** — the SW takes over and cleans up old caches.
- **`skipWaiting()`** — opt-out of waiting: "activate immediately, even with old tabs open."
  As a `GenerateSW` option: `skipWaiting: true`.
- **`clientsClaim()`** — "and take control of already-open tabs now, without a reload."
  Companion option: `clientsClaim: true`. Note: neither reloads the page; the tab keeps running
  the old JS until an actual refresh.
- **`updatefound` / `updated`** — lifecycle callbacks surfaced by `register-service-worker`:
  `updatefound` = new SW is downloading; `updated` = new SW finished installing (i.e. **a deploy
  happened** — this is our notification trigger).
- **`controllerchange`** — DOM event fired when the page's controller swaps (e.g. after
  `skipWaiting`). Used by some apps to auto-reload; we deliberately don't.

### Caching strategies (Workbox vocabulary)

- **Precache** — files downloaded and stored at install time, served straight from Cache
  Storage forever after; the network is never asked. Right for hashed assets, **wrong for
  `index.html`** (the root cause of our bug).
- **Precache manifest** — the build-generated list of `{url, revision}` entries the SW
  precaches. Changes to it are what make the browser see "a new SW version exists."
- **Runtime caching** — rules for requests *not* in the precache, declared as
  (`urlPattern` → `handler`) routes.
- **NetworkFirst** — try the network; fall back to cache only on failure/timeout. What we're
  adopting for HTML navigations.
- **CacheFirst** — serve from cache; hit the network only if missing. Fonts, images.
- **StaleWhileRevalidate** — answer from cache immediately, refresh the cache in the
  background. Slightly-stale-OK content.
- **NetworkOnly / CacheOnly** — no fallback variants.
- **Navigation request** — the request a browser makes when loading a page into a window
  (`request.mode === 'navigate'`) — i.e. the request for `index.html`. Our NetworkFirst route
  matches exactly these.

### Tooling

- **Workbox** — Google's service-worker toolkit; generates and powers our SW. We run **v4.3.1**
  (see [[02-tech-stack]] for v4 syntax gotchas).
- **`GenerateSW`** — Workbox webpack mode that writes the whole `service-worker.js` for you from
  config. What we use; no SW source file exists in the repo.
- **`InjectManifest`** — the alternative mode: you write your own SW file and Workbox injects
  the precache manifest into it. Only needed for custom logic (e.g. a `SKIP_WAITING` message
  listener) — out of scope short-term.
- **`register-service-worker`** — tiny Vue-team helper that registers the SW and exposes the
  lifecycle callbacks used in `src/registerServiceWorker.js`.

### App/deploy concepts

- **PWA (Progressive Web App)** — a site installable as an app (standalone window, dock/taskbar
  icon). Chromium's install criteria require HTTPS + a **web app manifest** + a registered
  service worker — the reason we can't simply delete the SW.
- **Web app manifest** — `manifest.json` (name, icons, colors, display mode) generated by
  `@vue/cli-plugin-pwa`. Unrelated to the *precache* manifest despite the name.
- **Content hashing** — `app.3f8a9c12.js`-style filenames; new content ⇒ new filename ⇒ safe to
  cache forever. Works today; the unhashed `index.html` is the weak link.
- **App shell** — the minimal HTML/JS/CSS needed to boot the UI; what precaching is meant for.
- **`Cache-Control: no-cache`** — "you may store it, but revalidate with the server before every
  use." Required on `index.html` and `service-worker.js` so updates are detected.
- **`Cache-Control: immutable`** — "never revalidate during max-age." Right for hashed assets.
- **`ChunkLoadError`** — runtime error when an old page lazy-loads a hashed chunk that a newer
  deploy deleted from the server. Mitigations in [[03-short-term-plan-update-notification]].

### Reload types (and what each actually clears)

| Action                                                              | HTTP cache                | Cache Storage                     | Service worker                                           | Result on our bug                                              |
| ------------------------------------------------------------------- | ------------------------- | --------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------- |
| **Normal reload** (`Cmd/Ctrl+R`, F5)                                | revalidates main resource | untouched                         | **intercepts the load**                                  | Old SW serves stale precached `index.html` → old app           |
| **Hard reload** (`Cmd/Ctrl+Shift+R`)                                | bypassed                  | untouched                         | **bypassed for this one load** (page loads uncontrolled) | New app for *one* load; old SW reclaims the next normal reload |
| **Empty Cache and Hard Reload** (right-click reload, DevTools open) | **cleared**               | untouched                         | bypassed for this one load, **stays registered**         | Same as hard reload — new app once, then back to old           |
| **Clear site data** (DevTools → Application, or browser settings)   | cleared                   | **deleted**                       | **unregistered**                                         | Actually fixes it (until the SW re-registers and re-precaches) |
| **Close every tab of the origin**                                   | —                         | old precache purged on activation | waiting SW **activates**                                 | New version from then on                                       |

The middle three rows explain the confusing "hard reload shows the new version, second refresh
goes back a version" case documented in [[01-current-state]]: hard reloads *bypass* the SW for
one navigation but never remove it or its Cache Storage.

## Browser support

### Service worker API itself

Mature, standardized, supported by **all browsers LiveCaller targets**:

| Browser                 | Since       | Notes                                                                                                         |
| ----------------------- | ----------- | ------------------------------------------------------------------------------------------------------------- |
| Chrome / Chromium       | 40 (2015)   | Reference implementation                                                                                      |
| Edge (Chromium)         | 79 (2020)   | Same engine as Chrome                                                                                         |
| Firefox                 | 44 (2016)   | Full support; **disabled in private windows**                                                                 |
| Safari (macOS)          | 11.1 (2018) | Full support for what we use                                                                                  |
| Safari (iOS)            | 11.3 (2018) | Supported; OS may evict Cache Storage after weeks of non-use (harmless for us — next visit just re-registers) |
| Brave / Opera / Vivaldi | —           | Chromium; same as Chrome                                                                                      |
| IE 11                   | never       | No SW; `register-service-worker` silently no-ops, app works as a plain site                                   |

Hard requirements everywhere: **HTTPS** (or `localhost`) and same-origin SW script.

Everything the short-term fix relies on sits well inside this baseline: Cache Storage, the
install/waiting/activate lifecycle, `skipWaiting`/`clientsClaim`, `request.mode === 'navigate'`
matching, and the Workbox 4 runtime all work in every browser above. We deliberately use **no**
bleeding-edge SW features (no navigation preload, no background sync, no push).

### Desktop PWA install ("Install app") — the part that constrains us

| Browser                       | Desktop install support                                                                    |
| ----------------------------- | ------------------------------------------------------------------------------------------ |
| Chrome / Edge / Brave / Opera | ✅ Full — install prompt in the omnibox/menu; requires manifest + SW (why the SW must stay) |
| Safari (macOS 14 Sonoma+)     | ⚠️ "Add to Dock" — works but does **not** require/consider a SW                            |
| Firefox (desktop)             | ❌ Removed in 2021 (no site-install support)                                                |

So the **installability requirement is effectively a Chromium requirement** — and Chromium is
what mandates the registered service worker. Users on Firefox/Safari still get the normal web
app (and after the fix, fresh versions on every refresh like everyone else).

### Practical takeaway

- The stale-cache bug affects **every modern browser equally** — it's our config, not a browser
  quirk.
- The fix (NetworkFirst navigations + `skipWaiting`/`clientsClaim` + update banner) behaves
  identically across Chrome, Edge, Firefox and Safari.
- The only browser-specific consideration is the install feature itself (Chromium-only), which
  the fix preserves untouched.

See also: [[01-current-state]], [[02-tech-stack]], [[03-short-term-plan-update-notification]].
