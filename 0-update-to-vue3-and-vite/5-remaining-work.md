# Remaining work — forward-looking only

State as of 2026-06-12. This report lists **only what is still left** to change, reduce or remove. Everything already migrated (Vue 3 + Vite 8, Pinia, the UI kit, all package majors, vue-api-query removal, module rewrites for profile/billing/blocked/live-conversations/dashboard, eslint 10 flat config, …) is documented in `4-remaining-legacy-stack.md` and intentionally not repeated here.

---

## A. Quick cleanup round ✅ (done 2026-06-12, except A5)

Results: **A1** ✅ `qs` removed (`entities.js` → `buildQuery`). **A2** ✅ the skeleton plugin turned out to be 100% dead (zero `PuSkeleton` usages — WidgetsListPage has its own CSS skeletons) → plugin folder + global registration deleted. **A3** ✅ `helpers.js`: `getTheDate`, `convertDateToTimeStamp`, `MakeQuerablePromise` + the dayjs/LiveCaller imports deleted. **A4** ✅ 113 orphaned images/icons deleted (images/ 1.4 MB → 660 KB); sass-boilerplate/fonts/`common.scss` correctly excluded (extension-less imports). **D1a** ✅ all 5 `darken()`/`lighten()` → precomputed hex with origin comments. **A6** ✅ lint pass: `--fix` flattened ~5 700 mechanical issues across ~400 files; `@typescript-eslint/parser` + `typescript` added so the chatbot/pbx-flow `lang="ts"` code is lintable for the first time; `vue/multi-word-component-names` disabled by policy. **Two real bugs fixed**: `ChangePassword.js` referenced the removed `showToast` (successful password change threw → error toast), and `LiveCaller.boot()` passed an undefined global `Vue` to booting callbacks. **A5** deferred (consolidate Cards when those pages are next touched).

Remaining lint findings (the follow-up sweep): 123 `no-unused-vars`, 37 `no-use-before-define`, 32 `max-len`, 9 `vue/no-mutating-props` (potential real bugs — review individually), 6 `vue/no-v-text-v-html-on-component`, 1 each `no-dupe-keys` / `no-constant-binary-expression` (suspicious — check first), plus ~78 warnings (`require-default-prop`, `require-explicit-emits`, `no-v-html`).

### Original A-table (for reference)

| #   | Item                           | What to do                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| --- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | `qs` dependency                | One call left: `src/stores/entities.js` (`qs.stringify({ filter: filters }, { encode: false })`). Replace with the in-house `buildQuery` from `src/shared/api/query.js` (or hand-built params) → drop `qs` from package.json                                                                                                                                                                                                                       |
| A2  | `vue-loading-skeleton` plugin  | Registered globally in `src/bootstrap.js` (`app.use(Skeleton)`) but used by a single page (`src/modules/widgets/WidgetsListPage.vue`). Import locally there, delete the global registration; if the plugin folder (`src/plugins/vue-loading-skeleton/`) has no other consumers, move its two components next to the widgets module or into `shared/`                                                                                               |
| A3  | `src/helpers/helpers.js` audit | Grab-bag module. Verify callers for `MakeQuerablePromise`, `getTheDate`, `convertDateToTimeStamp` (and anything else with 0 importers) and delete the dead ones. Long-term: dissolve the file into `src/shared/utils/`                                                                                                                                                                                                                             |
| A4  | Unused asset sweep             | `src/assets/images` = 235 files / 1.4 MB; several confirmed orphans from rewritten pages (`flag-icon.svg`, `padlock.svg`, `unlocked-icon.svg`, `search-icon.png`, …). Script it: list every file under `src/assets/`, grep the repo for each basename, delete unmatched. Watch out for dynamically-built paths (`/assets/images/devices/${name}.svg`, flags, os, browser, emoji-N) — whole directories referenced by template strings must be kept |
| A5  | Duplicate `Card` components    | `src/components/Card/index.vue` (`ag-card-*`; used by Integrations + Widgets pages) vs `src/components/objects/Card.vue` (used only by dashboard-classic StatsCards). Consolidate to one, ideally when next touching those pages                                                                                                                                                                                                                   |
| A6  | One `npm run lint` pass        | The new eslint 10 flat config hasn't been run across the repo yet; first run will surface dead vars/imports worth a mechanical sweep                                                                                                                                                                                                                                                                                                               |

## B. Last legacy page rewrites (real rounds)

| # | Item | Notes |
|---|---|---|
| ~~B1~~ | ~~**Customer detail page** `/customers/:id`~~ | ✅ (2026-06-12) → `src/modules/customer-profile/` (CustomerProfilePage + sections/: CustomerHeader, ConversationHistory, ConversationTabs, ChatArea, NotesSection, AttachmentsSection), all `<script setup>`, behavior kept. Old `views/pages/customers/` + `components/pages/customers/` deleted. Dead/broken code dropped rather than ported: `Information.vue` (rendered with `v-if="false"`), the table's checkbox/`checkAll`/`sortByName` machinery (referenced a `head(name)` column that never rendered; handlers threw). **Bugs fixed**: Notes delete used the removed vuejs-dialog `v-confirm` directive (was a no-op) → `useConfirm`; its delete URL used the visitor route id instead of the conversation id; note timestamps formatted `HH:MM` (months as minutes) → `HH:mm`; CustomerHeader mutated the `visitor` prop to seed custom fields → computed model; gallery event-bus listeners now unsubscribe on unmount (leaked before). Kept LcTable (not DataGrid) for the history list — its row-selection drives the right-hand panel and the active-row/deep-link (`conversation_id:` token) behavior; a DataGrid port would have changed UX |
| B2 | Error pages | `src/views/pages/errors/*` (401/403/404/419/500 + Loader), `src/components/pages/ErrorLayout.vue`, `src/components/Loader.vue` and the `src/plugins/error-page.js` plugin pattern. Small and self-contained; rewrite as a module + plain components, low priority |
| B3 | Layouts | `src/views/layouts/App.vue` + `MainLayout.vue` work fine; move under `src/modules/layout/` whenever convenient — cosmetic only |

## C. Product decisions (not tech debt)

| # | Item | Notes |
|---|---|---|
| C1 | Retire dashboard-classic | `/` (classic) and `/dashboard` (new module) both live, bridged by the TryNewVersion toggle. When the new dashboard becomes default, `src/modules/dashboard-classic/` + `stores/dashboard.js` delete cleanly |

## D. Ongoing background debt (chip away, no big-bang)

| # | Item | Notes |
|---|---|---|
| D1 | Options API → `<script setup>` | **Re-measured: the real number is 37 files**, not ~319 (most non-script-setup files are already Composition API via `setup()` — fine as-is). The 37 with true Options API (`data()`/`methods`): **5 are the customer-detail components** (die with B1 — do NOT convert separately); **8 are the cobrowse cluster** (`shared/cobrowse/*` + controls — the best standalone conversion round); 2 layouts (B3); 3 modals (`DeleteModal`, `DropConversationsModal`, `RefreshNotificationModal`); kit internals (`LcScrollbar`, `LcVSelect`, `DataGridBody`, `DataGridColumnSelector`, `TagCell`, skeleton-theme) which work fine and can wait; plus singles in inbox (3), pbx webphone (2), widgets `TagField`, `SettingsPage`, `ValidatedModelForm`, `FormInputComponent`, `Timezone` |
| D1a | Deprecated sass color functions | Exactly **5 occurrences** of `darken()`/`lighten()` left (TriggersPage.vue, ChannelLinesSelect.scss, ConversationReportsFilter.scss, FilterViewsSidebar.scss, ConversationTypeFilter.scss). 15-minute fix: `color.adjust(...)` or precomputed hex. Everything else is clean: 0× `::v-deep`, 0× `/deep/`, 0× global `map-get`, 0× `~` tilde imports |
| D1b | `@import` → `@use` in components | **85 `.vue` files** still use `@import` inside `<style>` — the reason `silenceDeprecations: ['import']` exists in vite.config. Most import self-contained partials (`sidepanel`, `filter-mixins`, local page `.scss`) and convert to `@use` mechanically; only files leaning on the boilerplate's shared `@import` scope need care. Finishing this sweep lets the silence flag be deleted before dart-sass 3 removes `@import` entirely. Hotspots: knowledge-base kb-settings tabs (7), triggers/editor (3), ai/workspaces (3), pbx (8 across trunks/queues/endpoints/flows) |
| D2 | `mixins/Translations.js` | Last global mixin (template `__()`); retiring it touches every template. Keep until a dedicated i18n round |
| D3 | Event bus | Intentionally kept (`LiveCaller.$on/$emit` → `shared/eventBus.js`) as the websocket fan-out backbone, ~250 call sites. Not debt — by design |

## E. Outside this repo (forks)

| # | Item | Notes |
|---|---|---|
| E1 | `@livecaller/sip.js` fork | Bump its `crypto-js` to ≥4.2 (audit critical) **and declare `events` in its dependencies** (the app currently ships the polyfill on the fork's behalf). Republish |
| E2 | `@livecaller/cobrowse-sdk` fork | Same `crypto-js < 4.2` pin → bump + republish. These two account for all 3 remaining `npm audit` criticals |

---

## Suggested order

1. **A-round** (one sitting): A1 + A2 + A3 + A4 + the 5 `darken()`/`lighten()` fixes (D1a), then A6 lint pass
2. **B1 customer detail rewrite** — the last legacy page; also erases 5 of the 37 Options-API files for free
3. **Cobrowse cluster** (8 files) — the best self-contained Options-API conversion round (D1)
4. **`@import` → `@use` sweep** (D1b) — mechanical, ends with deleting `silenceDeprecations: ['import']`
5. B2 error pages as a filler task
6. E1/E2 whenever fork publishing is on the table
7. C1 when product says the new dashboard is the default
