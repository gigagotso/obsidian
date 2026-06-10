# Package Removability Audit — "Do we even need it?"

> Companion to **[[PHP 7.4 to 8.x & Laravel Upgrade Plan]]** and **[[Package Upgrade Reference (per-package)]]**. Every package was grepped against the actual codebase (`app/`, `config/`, `routes/`, `database/`, `resources/` — excluding `vendor/`) to decide KEEP vs REMOVE *before* spending effort upgrading it. **The cheapest upgrade is the one you delete.**

---

## TL;DR — what to remove

**6 production packages are dead weight or near-dead** and should be dropped/migrated-off rather than upgraded:

| Package | Finding | Action | Effort |
|---|---|---|---|
| `beyondcode/laravel-websockets` | replaced by Sockudo (already known) | **REMOVE** | trivial |
| `laravel/ui` | zero usage — no `Auth::routes()`, no scaffolding, no views. API backend doesn't need it | **REMOVE** | trivial |
| `barryvdh/laravel-translation-manager` | only a dead `config/translation-manager.php`; no routes/models/usage | **REMOVE** | trivial |
| `eduardokum/laravel-mail-auto-embed` | no mailables, no `@embed`, no references anywhere | **REMOVE** | trivial (and skips the 1→2 Symfony-Mailer rewrite!) |
| `inspheric/nova-defaultable` | zero usage in any Nova resource | **REMOVE** | trivial |
| `kutia-software-company/larafirebase` | zero FCM/push usage in code | **REMOVE** | trivial |

**Plus dev tools that are genuinely unused** (no code references / no `ray()` calls):

| Package | Action |
|---|---|
| `beyondcode/laravel-dump-server` | **REMOVE** (superseded by built-in / Herd / Ray) |
| `spatie/laravel-ray` | **KEEP** — team uses Ray for debugging. Bump within `^1` (already spans L7→L13). |
| `fzaninotto/faker` | **REPLACE** → `fakerphp/faker` (abandoned) |
| `barryvdh/laravel-debugbar`, `barryvdh/laravel-ide-helper` | **TEAM DECISION** — no code refs (expected for dev tools); keep+bump if devs use them locally, else drop |

**1 optional migrate-off:**

| Package | Finding | Action |
|---|---|---|
| `spatie/laravel-tags` | only the `HasSlug` trait is used; the actual tagging is a **custom** `app/Models/Traits/HasTags.php`. The package's morphMany tagging is reinvented locally | **MIGRATE-OFF (optional)** — replace `HasSlug` (~4–5 sites: Tag model + slug migration) with `Str::slug()`, then drop the dependency and skip its v2→v3→v4 upgrade chain |

> **Net:** removing these eliminates **~8 packages** from the upgrade surface, including one (`mail-auto-embed`) that would otherwise need a full 1→2 rewrite for the Swiftmailer→Symfony Mailer switch.

---

## ⚠️ Correction to earlier research — `consoletvs/charts` is USED, do NOT just drop it

The first research pass flagged `consoletvs/charts` as "abandoned → replace with Nova Metrics." The usage audit corrects this:

- It is **actively used** — `app/Charts/Chart.php`, `ConversationsChart.php`, `ConversationReviewsChart.php` (extend `ConsoleTVs\Charts\Classes\Highcharts\Chart`), instantiated in `app/Actions/Reports/GetConversationsDetailedReport.php` and rendered in `SendWeeklyReportsToAccounts.php`. Default lib is **Highcharts** (`config/charts.php`).
- These are **Highcharts rendered into emailed/PDF reports — NOT Nova dashboard cards.** So **Nova Metrics is the wrong replacement.**
- It is **not a blocker**: the v6 line still gets Laravel-compat bumps (6.9.0 supports Laravel 12). **Recommendation: KEEP `consoletvs/charts` on `^6.x` through the whole upgrade** (it installs fine on L8/9/10), and only revisit (→ Larapex Charts or a maintained Highcharts wrapper) as a separate, post-upgrade cleanup if desired. Don't let it gate the Laravel work.

---

## KEEP — confirmed actively used (the real upgrade surface)

These are genuinely used and must be upgraded per **[[Package Upgrade Reference (per-package)]]**:

**Auth (all three are needed — different purposes, no consolidation benefit):**
- `laravel/passport` — **primary API OAuth2** for app users (`api` guard, `HasApiTokens` on User, custom Client/Token models, oauth routes). KEEP.
- `tymon/jwt-auth` — **stateless `widget` guard** for anonymous widget visitors (`Visitor implements JWTSubject`, `routes/widget.php` uses `auth:widget`). Different use case from Passport. KEEP.
- `firebase/php-jwt` — **3rd-party token handling**, independent of the above: validates widget-SDK JWTs (`ValidJWTSignature`) and mints/verifies **LiveKit** tokens (`LiveKitTokenService`). KEEP. ⚠ Note this ties into the Web-Call→LiveKit work.

> Auth verdict: **do not consolidate.** Passport (expiring tokens/scopes for staff) + tymon (long-TTL stateless for visitors) + firebase (external token verify) each serve a distinct flow.

**Core feature packages (heavily integrated):**
- `spatie/laravel-medialibrary` — 12 models, 67+ imports. KEEP (highest-risk upgrade, see per-package doc).
- `spatie/laravel-schemaless-attributes` — `extra_attributes` on 30+ models, 20+ migrations. KEEP (foundational).
- `spatie/laravel-query-builder` — every list endpoint + custom filter ecosystem. KEEP.
- `spatie/laravel-permission` — RBAC backbone (`HasRoles`, Role/Permission models). KEEP.
- `spatie/laravel-webhook-client` — PBX + Social Hub webhook intake. KEEP.
- `spatie/opening-hours` — queue/department business hours. KEEP.
- `spatie/eloquent-sortable` — Tag/Queue/AccountField ordering. KEEP.
- `lorisleiva/laravel-actions` — **53 Action classes on the 0.x base-class API.** KEEP the package, but the upgrade is **MEDIUM effort, not contained** — see the dedicated section below.
- `staudenmeir/eloquent-json-relations` — JSON-path relations (`extra_attributes->widget_id`) on Conversation/Widget/ConversationMessage/Kb. KEEP.
- `chelout/laravel-relationship-events` — BelongsToMany events for User↔Queue sync. KEEP.
- `maatwebsite/excel` — 19 export classes (analytics/reports). KEEP.
- `pusher/pusher-php-server` — broadcast auth / presence / webhook signatures for **Sockudo**. KEEP (bump 4→7).
- `marcelog/pami` — **Asterisk AMI** integration (queues, SIP reloads, call control). KEEP (critical telephony).
- `torann/geoip` — `geoip()` IP geolocation across security logs/visitor profiles. KEEP.
- `geoip2/geoip2` — used **directly** for MaxMind ASN/threat-intel lookups (separate from torann). KEEP.
- `monicahq/laravel-cloudflare` — app **is** behind Cloudflare; `TrustProxies` + daily `cloudflare:reload`. KEEP.
- `mews/purifier` — XSS sanitization via `CleanStringInput`/`CleanJsonInput` casts on 6+ models. KEEP.
- `laravel/cashier` — **fully implemented billing** (Billable on Account, 4 billing controllers, Stripe customer sync, subscription seat limits). KEEP.
- `laravel/helpers` — used for `array_first()` / `str_is()` in ~5 files. KEEP, *or* migrate those few sites to `Arr::first()`/`Str::is()` and drop it (trivial, optional).

**Nova add-ons that ARE used (so they gate Nova 4):**
- `dillingham/nova-attach-many` — used in `Role` + `Widget` resources. KEEP → `^2.0` (Nova 4).
- `koss-shtukert/...select2-auto-complete` — used in **1 site** (`Account` timezone field). The Nova-4 migration is small: replace that one `Select2::make` with a native `Select::make(...)->searchable()`. Then **REMOVE** the package (it's dead/Nova-3-only anyway).
- `titasgailius/search-relations` — base trait on `app/Nova/Resource.php` (`$searchRelations` in Role/Widget). KEEP → `^2.0`.

**Monitoring/ops (safe defaults, keep):**
- `laravel/telescope` — prod-guarded (`night()`, error-only filter, `@livecaller.io` gate). KEEP.
- `laravel/horizon` — dashboard gated to `@livecaller.io`; queue is `sync` by default. KEEP (harmless UI).
- `inspector-apm/inspector-laravel` — disabled by default (`INSPECTOR_ENABLE=false`), IP-gated middleware. KEEP (opt-in).

---

## Deep-dive: `lorisleiva/laravel-actions` — the hidden medium-effort item

**Installed: v0.3.0 (the old base-class API). This is a forced rewrite at Laravel 9, and it's bigger than it first looks.**

What the code actually does (verified):
- **53** classes, all `extends Lorisleiva\Actions\Action`; **44** define a `handle()` method (0.x calls `handle()` internally, you call `->run()`).
- **45** files call `->run()`; **25** use `->actingAs($user)->run()`.
- **45** define `authorize()` and/or `rules()` — i.e. they rely on the package's **built-in validation + authorization**.
- **32** use the **attributes bag** (`protected $attributes = [...]`, `$this->get()`, `fill()`, `$this->user()`).
- **0** used as controllers (no route references) or as queued jobs.

**Why this matters:** the 2.x rewrite *removed* exactly these features — the attributes bag, automatic `authorize()`/`rules()` on direct calls, `$this->user()`, and `actingAs()`. So a straight bump breaks most of the 53 classes. The version wall is hard: **1.x is the last line with the old API and it stops at Laravel 8; Laravel 9 requires 2.x.** There is no "keep the old API on L9" option.

### Recommended staged path

1. **Laravel 8 step — bump `0.3 → ^1.2.2` (tiny change, same API).** 1.2.2 keeps `extends Action` + `authorize()`/`rules()`/`$attributes`/`user()`/`actingAs()`/`->run()` and runs on PHP 7.4 + Laravel 8. Verifies nothing else broke. Buys time. *(Sanity-check: 0.3.0 may even run on L8 as-is; either way land on 1.2.2 before L9.)*

2. **Laravel 9 step — the rewrite to `^2.7`.** Lowest-churn 2.x migration, per class:
   - Replace `extends Action` with `use Lorisleiva\Actions\Concerns\AsAction;` (keep `handle()` as-is). Call sites change `(new X())->run()` → `X::run()`.
   - **Attributes bag (32 classes):** add the `WithAttributes` trait (2.1+) to restore `$this->set()/get()/fill()/$attributes` — this minimizes churn. *Or* refactor `$attributes` into explicit `handle(...)` parameters (cleaner, more work).
   - **`authorize()`/`rules()` (45 classes):** these no longer auto-run on `::run()` (they only fire via the `asController` decorator, which you don't use). Choose per action:
     - move the validation to the **calling controller's FormRequest** (best where the action is called from an HTTP controller), **or**
     - call `$this->validate()` / a manual `Validator` explicitly at the top of `handle()` using the existing `rules()` array.
   - **`$this->user()` / `->actingAs()` (25 sites):** removed in 2.x. Pass the user explicitly — change `->actingAs($u)->run($args)` to `X::run($u, $args)` and accept `$user` as the first `handle()` parameter (or store via `WithAttributes`).
   - ⚠ Register actions in a service provider if any later use `routes()` (none do today).

**Effort estimate:** ~53 classes touched, but mechanical and patternable. Budget it as a distinct sub-task of the L9 phase (alongside Nova 4 + medialibrary). It's the **3rd-biggest code item** after Nova 4 and medialibrary.

**Alternative considered — drop the package entirely:** since these are just invokable service objects, they *could* become plain `__invoke()` classes. Rejected: it's equal-or-more work and you'd lose the validation/authorization scaffolding. Migrating to 2.x via `AsAction` + `WithAttributes` is the better mapping.

---

## Impact on the plan

- **Phase 0 grows** from "remove websockets + swap faker" to also dropping: `laravel/ui`, `barryvdh/laravel-translation-manager`, `eduardokum/laravel-mail-auto-embed`, `inspheric/nova-defaultable`, `kutia/larafirebase`, `beyondcode/laravel-dump-server`, `spatie/laravel-ray` — all zero-risk deletes doable today on Laravel 7 / PHP 7.4.
- **The L9 step gets lighter:** dropping `eduardokum/laravel-mail-auto-embed` removes a 1→2 rewrite; dropping `barryvdh/laravel-translation-manager` removes a PHP-8.1-floor blocker.
- **The Nova 4 step gets slightly lighter:** `nova-defaultable` is just deleted; `select2` collapses to one native-field swap.
- **`consoletvs/charts` is reclassified KEEP** (`^6`) — not a blocker, revisit post-upgrade.
- **Optional extra cleanup:** migrate off `spatie/laravel-tags` (HasSlug-only) and `laravel/helpers` (~5 sites) to shed two more upgrade targets.
