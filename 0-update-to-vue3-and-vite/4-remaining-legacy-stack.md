# Remaining legacy stack — what to remove or update

State as of 2026-06-12, after: Vue 3 (no compat) + Vite migration, Bootstrap/Vuex/vuejs-dialog removal, Vue Flow port, navigation rework, DataGrid layout normalization, profile module (`/personal-details`, `/notifications`), billing module (`/settings/billing/*`), and deletion of the dead `/music` + `/stats` pages.

---

## 1. Legacy pages still routed (rewrite as `src/modules/`)

| Page | Route | Source | Status |
|---|---|---|---|
| Customer detail | `/customers/:id` | `src/views/pages/customers/_id.vue` + 7 components in `src/components/pages/customers/` (ChatArea, Comments, CustomerAttachments, CustomerHeader, CustomerTable, CustomerTabs, Information) | **next up** |
| ~~Dashboard~~ | `/` | ✅ → `src/modules/dashboard-classic/` (DashboardClassicPage + sections/: StatsCards, ConversationTables, ChartsSection, AgentsPanel, EndpointStatus, TopUsersChart, ComparisonChart), all `<script setup>`; behavior kept as-is. Bonus fixes: Agents' broken `import { LiveCaller }` (imported the class, not the instance — dropdown serving-counts always threw) and chart instances now destroyed on unmount. Old `views/pages/Dashboard.vue` + `components/pages/dashboard/` deleted; the inline agents SVG moved to `assets/images/dashboard-agents-icon.svg` |
| Error pages | 401/403/404/419/500 | `src/views/pages/errors/` + `src/components/pages/ErrorLayout.vue` | small, low priority |

Everything else already lives under `src/modules/`.

## 2. Mixins → composables ✅ (done — 1 left, intentional)

- Deleted: `Toast`, `DisplayErrorMessage`, `ParseName` (was unused), `CheckImage` (→ `src/shared/utils/files.js` `isFileImage`), `Shortcuts` (was a dead import in DropConversationsModal), `CustomFields` + `Table` (inlined into their single legacy consumers, retire fully with the customers rewrite), plus earlier `Billing` + `InteractsWithDates`
- All toast call sites (incl. silent `proxy.showToast?.()` in conversation-windows and `@error="displayErrorMessage"` template handlers) now go through `useToast`
- Remaining: `Translations` (`__()`) — kept as the template-facing global, now a thin delegate to `src/shared/i18n.js` `translate()`; retiring it means touching every template that calls `__()`

## 3. Global event bus ✅ (consolidated — kept by design for realtime)

What was done:
- `src/util/EventBus.js` deleted; `LiveCaller.$on/$once/$off/$emit` now delegate to the single modern bus in `src/shared/eventBus.js` (which gained array-of-events support)
- `collapseMenu` moved to Pinia: new `src/stores/ui.js` (`menuCollapsed`); AppNavigation writes, MainLayout reads via computed
- `validation-error` emit in `src/util/axios.js` removed (had zero listeners)
- `window.eventBus` global removed; the error-page plugin imports the bus directly

What stays (intentional): the bus itself. ~250 call sites across 60 files use `LiveCaller.$on('ws:…')` / per-conversation channels (`conversations.<id>.…`, `openGallery<id>`, cobrowse, SIP) — it's the websocket fan-out backbone, a legitimate pub/sub use, now backed by one small implementation. Converting those to Pinia would be a separate large project with little payoff.

## 4. Old libraries still imported (done ✅ except highcharts)

Removed ✅:
- `form-backend-validation` — `ValidatedModelForm` keeps its `{form, submit}` slot API but runs on an in-house `src/components/Form/Form.js` (also killed the bundled vulnerable axios 0.32 it dragged in)
- `collect.js` — replaced with `pick()` in `src/shared/utils/object.js` / spreads
- `vue-multiselect` — `CheckBoxSelect.vue` rewritten as a self-contained searchable checkbox multi-select (search + infinite scroll + "All" option preserved)
- `linkifyjs` 2 → **4** (+ `linkify-html`); the `v-linkify` directive keeps `target: '_blank'` explicitly
- `vue-api-query` + `src/models/` (all 9 files) — queries now hit `LiveCaller.request` directly with the same Spatie query-string conventions, built by the small `buildQuery()` in `src/shared/api/query.js` (`filter[k]`, `include=`, `fields[rel]=`, `sort=`, `page`/`limit`). `UserStatus` moved to `src/shared/userStatus.js` (7 importers updated); the auth user is now a plain object from a `createUser()` factory in `src/stores/auth.js` that keeps the two behaviors the app used (`can()`, `updateStatus()`). `conversation.messages()` relation queries became explicit `/conversations/:id/messages` GETs; `new Conversation/ConversationMessage(data)` wrappers became plain objects

Still in:
| Library | Usage | Action |
|---|---|---|
| `highcharts` 9 | dashboard Diagram/VerticalDiagram + 3 analytics components | decide: upgrade to 13 vs replace; do with the dashboard round |
| `register-service-worker`, `nprogress` | one import each, all functional | keep |

Also removed (2026-06-12 small-deps audit): `detectrtc` (unmaintained; replaced with native `enumerateDevices` + Permissions API in `useMicrophone.js`), `crypto-js` (its only importer was the dead `base64Url()` helper — deleted; the audit criticals are the forks' own nested copies, unaffected either way), `linkifyjs` as a direct dep (nothing imports it directly — it stays as `linkify-html`'s transitive dep), `@ckpack/vue-color` (`components/Colorpicker` rewritten on a native `<input type="color">` — the old wrapper hid the Chrome picker's alpha/field controls anyway, same approach as the tag form's palette).

## 5. Outdated packages ✅ (done except the sip.js forks)

All done in the 2026-06-12 round — **tested on Node 24, everything works**:

- **Sentry 10 / vite-plugin 5** — `new Sentry.BrowserTracing()` → `Sentry.browserTracingIntegration()` in `src/bootstrap.js`; plugin options (`sourcemaps.assets`) and the `SENTRY_PROPERTIES` flow unchanged
- **eslint 10 + eslint-plugin-vue 10** — flat config (`eslint.config.mjs`), `.eslintrc.js`/`.eslintignore` deleted. The airbnb chain was DROPPED, not upgraded: `@vue/eslint-config-airbnb@8` (latest) still peers on eslint 8 / plugin-vue 9 — dead end for flat config; most of its opinions were locally overridden anyway. Formatting rules removed from eslint-core live in `@stylistic/eslint-plugin`. Also gone: `@babel/core`, `@babel/eslint-parser`, `eslint-plugin-babel`, `eslint-plugin-import`, `eslint-config-airbnb-base`. Lint script is now plain `eslint src`
- **vite 8 / @vitejs/plugin-vue 6 / vite-plugin-pwa 1** — config-compatible; only the now-obsolete `css.scss.api: 'modern'` flag dropped (modern is the only sass API left)
- **highcharts 13** — modules auto-register on import since v12: `import exportingInit from 'highcharts/modules/exporting'; exportingInit(Highcharts)` → side-effect `import 'highcharts/modules/exporting'` (5 components)
- **@stripe/stripe-js 9** — drop-in; `loadStripe` + split card elements + `confirmCardSetup` all still supported
- **pusher-js 8 + official `laravel-echo` 2** — the `@livecaller/laravel-echo` fork is REMOVED, replaced by the original package. Echo 2 doesn't bundle the client: the `Pusher` class is passed via options in `src/util/Echo.js` (+ `withoutInterceptors: true`, we register our own axios interceptor). `PusherBatchAuthorizer` needed no changes — pusher-js 8 proxies the legacy `authorizer` option into `channelAuthorization.customHandler`
- **vue-router 5.1** — no code changes needed (no deprecated API usage; the `getMatchedComponents` helper already works off `to.matched`)

Fallout from the clean install — the old node_modules tree had been silently satisfying undeclared dependencies:

- `postcss.config.js` required `autoprefixer` that was never in package.json (rode in via the vue-cli-era toolchain) → rewritten as ESM `postcss.config.mjs`, `autoprefixer` declared as devDep, `.browserslistrc` modernized (`not ie <= 8` → `not dead`)
- `@livecaller/sip.js` does `require("events")` (Node's EventEmitter) without declaring it → the browser polyfill `events@3` is now an explicit app dependency. **When bumping crypto-js in the fork (§6), also add `events` to the fork's own dependencies.**
- pusher-js 8 throws `Options object must provide a cluster` even for self-hosted ws servers → `cluster: ''` added in `src/util/Echo.js` (same trick laravel-echo's reverb path uses)

Still pinned (intentional): `sip.js` / `@livecaller/sip.js` forks — only touch with telephony testing.

Earlier rounds ✅: axios 1.17 (was 0.18), jwt-decode 4, pinia 3, sass/qs in-range. (`vuedraggable` "latest 2.24" is npm-tag noise — 4.x is correct for Vue 3.)

## 6. Security audit (34 findings: 4 critical, 8 high)

- **Runtime critical:** `crypto-js < 4.2` (weak PBKDF2) pinned inside the private forks `@livecaller/cobrowse-sdk` and `@livecaller/sip.js` → fix by bumping crypto-js in the forks and republishing. Not fixable from the app.
- ~~Most of the rest live in the dev toolchain (eslint 7 stack)~~ ✅ cleared by the eslint 10 migration — `npm audit` is now down to the 3 crypto-js criticals inside the forks.

## 7. Style / code debt

- ~~`::v-deep` sweep~~ ✅ — all 34 remaining files converted to `:deep()` (inline usages wrapped directly; `X ::v-deep { … }` wrapper blocks restructured with `:deep()` per child rule, recursing through `@media`/`@include`); every transformed style compile-checked with the sass CLI
- ~80 components under `src/components/`, `src/views/`, `src/shared/` still on Options API → convert opportunistically as each area is touched (don't big-bang)
- ~~`src/assets/sass-boilerplate/` audit~~ ✅ — rewritten as `@use`/`@forward` modules: breakpoint-sass vendor deleted (replaced by a 20-line `mq()`), unused mixins/functions removed (`center`, `flex-end`, `fonts`, `parentState`, `map-deep-set`/`depth`/`has-keys`/`zip`), `size()`'s never-defined `is-length` validation dropped, `darken()` → precomputed hex; Vite on the modern sass API; ~60 redundant per-file boilerplate `@import`s deleted (additionalData already injects it). Only the transitional `@import` deprecation is silenced in vite config (additionalData members must reach styles that `@import` local companion files)
- `getCurrentInstance()` uid generation in 5 kit components — acceptable, no action

## Suggested order

1. ~~Profile pages~~ ✅ → ~~Billing~~ ✅ → ~~Dashboard~~ ✅ (`dashboard-classic` module) → **Customer detail** (last legacy page rewrite)
2. Quick wins alongside: `npm audit fix`, collect.js removal, linkifyjs 4, `::v-deep` sweep
3. Mixins → composables (mostly falls out of the page rewrites)
4. ~~eslint 7 → 10, Sentry 10, Vite 8~~ ✅ (full §5 round done — incl. highcharts 13, stripe 9, pusher 8 + official laravel-echo, vue-router 5)
5. ~~Event bus consolidation~~ ✅; ~~vue-api-query removal~~ ✅; highcharts decision
6. Forks: crypto-js bump in cobrowse-sdk/sip.js; pusher-js 8 + laravel-echo together
