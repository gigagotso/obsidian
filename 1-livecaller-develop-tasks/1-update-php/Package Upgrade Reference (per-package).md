# Package-by-Package Upgrade Reference

> Companion to **[[PHP 7.4 to 8.x & Laravel Upgrade Plan]]**. This is the in-depth, per-package research — exact version pins at each Laravel milestone, breaking changes, and concrete migration steps. Versions verified against Packagist, GitHub releases/upgrade guides, and Laravel Shift compatibility tables.

**Framework floors (reference):** L8 = PHP ≥7.3 · L9 = PHP ≥8.0.2 · L10 = PHP ≥8.1. Target end-state: **Laravel 10 + PHP 8.2 + Nova 4.**

---

## ⚠️ Read this first — PHP-floor traps

These packages' *latest* versions silently require **PHP 8.1+** and will break `composer install` while you are still on PHP 7.4. **Cap/pin them** until the PHP bump, then let them float:

| Package | Trap | Hold until |
|---|---|---|
| `aws/aws-sdk-php` | latest 3.384+ needs PHP ≥8.1 — `^3.300` resolves to a build that breaks on 7.4 | cap `<3.350` on PHP 7.4 |
| `geoip2/geoip2` | 3.x needs PHP ≥8.1 | stay `^2.x` on 7.4 |
| `barryvdh/laravel-translation-manager` | 0.6.x needs PHP ≥8.1 | stay `0.5.8` on 7.4 |
| `chelout/laravel-relationship-events` | 2.x needs PHP ≥8.1 | stay `^1.x` on 7.4 |
| `staudenmeir/eloquent-json-relations` | 1.15 needs PHP ≥8.3 | pin exact minor per Laravel |
| `spatie/eloquent-sortable` | 4.x needs PHP ≥8.1 | may force PHP 8 early — see below |

**Two real rewrites — budget the most time:** `lorisleiva/laravel-actions` (0.x → 2.x) and `eduardokum/laravel-mail-auto-embed` (1 → 2, Swiftmailer→Symfony Mailer). *Update: the usage audit found `mail-auto-embed` is **unused** → removed in Phase 0, so its rewrite is moot. See [[Package Removability Audit]].*

> **Removed in Phase 0 (verified unused) — ignore their rows below:** `laravel/ui`, `barryvdh/laravel-translation-manager`, `eduardokum/laravel-mail-auto-embed`, `inspheric/nova-defaultable`, `kutia/larafirebase`, `beyondcode/laravel-dump-server`, plus `beyondcode/laravel-websockets`. **`spatie/laravel-ray` is KEPT** (team uses it — bump within `^1`). `koss-shtukert/select2` is removed at the Nova 4 step (1 native-field swap).

---

## Per-milestone version pin table (master cheat-sheet)

| Package | Now | L8 (PHP 7.4→8.1) | L9 (PHP 8.0+) | **L10 (PHP 8.2)** |
|---|---|---|---|---|
| **laravel/framework** | 7.30 | `^8.0` | `^9.0` | `^10.0` |
| **laravel/nova** | 3.0.x-dev | `3.31`/`4` | `3.31`/`4` | **`^4` (PAID)** |
| laravel/cashier | 12.7 | `^13` | `^14` | `^15` |
| laravel/horizon | 4.3 | `^5` | `^5` | `^5` |
| laravel/passport | 9.3 | `^10` | `^11` | `^11.5` / `^12` |
| laravel/telescope | 3.5 | `^5` | `^5` | `^5` |
| laravel/ui | 2.5 | `^3` | `^4` | `^4.2` |
| laravel/tinker | 2.4 | `^2.7` | `^2.7` | `^2.7` |
| laravel/helpers | 1.2 | `^1` | `^1` | `^1` (or drop) |
| spatie/laravel-medialibrary | 7.19 | `→8→9` | `^10` | `^10` |
| spatie/laravel-permission | 3.18 | `^5` | `^5` | `^5` / `^6` |
| spatie/laravel-query-builder | 2.8 | `^4` | `^5.7` | `^5.7` / `^6` |
| spatie/laravel-schemaless-attributes | 1.7 | `1.8`/`2.3` | `^2.3` | `^2.4` |
| spatie/laravel-tags | 2.6 | `^3` | `^4` | `^4` |
| spatie/laravel-webhook-client | 2.7 | `^3` | `^3` | `^3` |
| spatie/opening-hours | 2.7 | `^3` | `^3` | `^4` (PHP 8.2) |
| spatie/eloquent-sortable | 3.8 | `^4` (PHP 8) | `^4` | `^4` / `^5` |
| doctrine/dbal | 2.12 | `^2` | `^2`/`^3` | **`^3`** |
| guzzlehttp/guzzle | 6.5.8 | `^7.5` | `^7.5` | `^7.5` |
| guzzlehttp/psr7 | 1.9 | `^2` (via guzzle) | `^2` | `^2` |
| firebase/php-jwt | 5.2 | `^6.10` | `^7.0` | `^7.0` |
| tymon/jwt-auth | 1.0.2 | `2.0` (after PHP 8) | `^2.0` | `^2.0` |
| league/flysystem-aws-s3-v3 | ~1.0 | `~1.0` | **`^3.0`** | `^3.0` |
| fideloper/proxy | 4.3 | keep | **remove** | — |
| fruitcake/laravel-cors | 2.0 | keep | **remove** | — |
| lorisleiva/laravel-actions | 0.3 | `^1.2.2` | `^2.7` | `^2.7` (`^2.10` PHP 8.2) |
| maatwebsite/excel | 3.1.26 | `^3.1` | `^3.1` | `^3.1` (latest) |
| mews/purifier | 3.3 | `^3.4` | `^3.4` | `^3.4` |
| chelout/laravel-relationship-events | 1.2 | `^1` | `^1` | `^2` (PHP 8.1) |
| staudenmeir/eloquent-json-relations | 1.4 | `1.x` | `1.7` | `1.8` |
| eduardokum/laravel-mail-auto-embed | 1.0 | `^1` | **`^2`** | `^2` |
| kutia/larafirebase | 1.0 | `^1.3` | `^1.3` | `^1.3` (dormant) |
| monicahq/laravel-cloudflare | 1.4 | `^2`/`^3` | `^3` | `^3` |
| torann/geoip | 1.2 | `^1` | `^3` (after PHP 8) | `^3` |
| geoip2/geoip2 | 2.10 | `^2` | `^3` (PHP 8.1) | `^3` |
| aws/aws-sdk-php | 3.300 | cap `<3.350` | float | float |
| pusher/pusher-php-server | 4.1 | `^7.2` | `^7.2` | `^7.2` |
| **dev:** facade/ignition | 2.0 | `^2.5` | **spatie/laravel-ignition `^1`** | `^2` |
| **dev:** nunomaduro/collision | 4.1 | `^5` | `^6` | `^6` |
| **dev:** phpunit/phpunit | 8.5 | `^9.5` | `^9.5` | `^9.6` (or `^10.1`) |
| **dev:** mockery/mockery | 1.3 | `^1.6` | `^1.6` | `^1.6` |
| **dev:** barryvdh/laravel-debugbar | 3.2 | `^3.9` | `^3.9` | `^3.9` |
| **dev:** barryvdh/laravel-ide-helper | 2.6 | `^2.13` | `^2.13` | `^2.13` |
| **dev:** fzaninotto/faker | 1.4 | **fakerphp/faker** | — | — |
| **dev:** beyondcode/laravel-dump-server | 1.0 | `^2.1` or drop | — | — |
| **dev:** spatie/laravel-ray | 1.43 | `^1` | `^1` | `^1` |
| **REMOVE NOW:** beyondcode/laravel-websockets | 1.4 | gone (Sockudo) | — | — |
| **REPLACE:** consoletvs/charts | 6.5 | deprecated → Nova Metrics / Larapex | — | — |
| **REPLACE:** inspheric/nova-defaultable | 1.2 | → Nova `->default()` | — | — |
| **REPLACE:** koss-shtukert/...select2 | 1.21 | → Nova `->searchable()` | — | — |
| dillingham/nova-attach-many | 1.0 | — | — | `^2.0` (Nova 4) |
| titasgailius/search-relations | 1.0 | — | — | `^2.0` (Nova 4/5) |

---

# 1. Laravel first-party

### laravel/nova — CRITICAL, the gating dependency
- **Version map:** Nova 3 supports L6–L9 (L8 needs ≥3.10.0, L9 needs ≥3.31.0). **Nova 4 = L8/9/10.** Nova 5 drops L8/9 and needs PHP 8.1+ (only relevant past L10).
- **PAID upgrade.** Nova 3 license does **not** entitle you to Nova 4 — budget the purchase + an `auth.json` credential before the L10 cutover. Single biggest cost/risk item.
- **Vue 2 → Vue 3 + Inertia + Tailwind 2 rewrite.** Every custom field/card/tool/dashboard component (you have 29 custom files + `nova-components/`) must be ported: Vue 3 component registration, removal of Vue 2 filters/event-bus, re-scaffold with Nova 4 generators, replace any Vue-2-only JS libs, Tailwind 1→2 classes.
- **Resource-API breaking changes (3→4):**
  1. `NovaRequest` is now required in signatures of `fields()`, `fieldsForIndex/Detail/Create/Update()`, `cards()`, `filters()`, `lenses()`, `actions()`; filter `apply()`/`options()`; action `fields()`. Change `Request` → `Laravel\Nova\Http\Requests\NovaRequest` everywhere.
  2. **Dashboards are now classes** — cards in `NovaServiceProvider` move into `Dashboard` classes (default `Main`).
  3. `config/nova.php` gains `middleware`, `api_middleware`, `storage_disk` keys.
  4. New menu/notifications APIs + updated authorization flow.
- **Republish:** `php artisan vendor:publish --tag=nova-assets --force` and `--tag=nova-lang --force`; reconcile `config/nova.php`.
- **Sequencing:** keep Nova 3 through L8 + L9 (bump 3.10 → 3.31), then buy + migrate to Nova 4 as part of the L9→L10 step (mandatory for L10).

### laravel/cashier (Stripe) → 13 → 14 → **15**
- 12.x = L6–8; **13.7.0** added L9; 14.x = L9/L10; **15.x** = L10/11 (PHP 8.1+). Land on 15 at L10.
- Breaking: Stripe SDK + pinned Stripe API version bump each major (review webhooks + direct `Stripe\*` calls); `Billable` casts/columns added (`pm_type`, `pm_last_four`, `trial_ends_at` — verify billable table); Stripe Tax / invoice-rendering changes; PHP 8.1 floor at 15.
- Republish `config/cashier.php`; re-check Cashier migrations.

### laravel/horizon → **5** (one jump)
- 5.x spans L8–L13 (PHP 8.0+). Jump 4→5 once at L8.
- Breaking: front-end/asset rewrite — **`php artisan horizon:publish`** after each upgrade; new config keys (`silenced`, `trim`, `fast_termination`, tuned `waits`/`memory_limit`); verify supervisor `environments` config.

### laravel/passport → 10 → **11** (or 12)
- 10.x = L8; 11.x = L9/L10 (≥11.5 for L10); 12.x also L10.
- Breaking: add migration for **`provider` column on `oauth_clients`**; **remove `Passport::routes()`** from provider (auto-registered now); optional `Passport::hashClientSecrets()`; `lcobucci/jwt` v4→v5 at 12.x.
- ⚠️ **Shares `lcobucci/jwt` with tymon/jwt-auth — align versions** (most likely composer-resolution conflict in the whole upgrade).
- Republish `config/passport.php` / `passport-views` if customized.

### laravel/telescope → **5** (one jump)
- 5.x spans L8.37–L13 (PHP 8.0+). Jump straight to 5 at L8.
- Breaking: re-publish assets each upgrade (**`php artisan telescope:publish`**); `telescope_entries` schema changes (`php artisan migrate`); verify `gate()`/`filter`/`tag` closures.

### laravel/ui → 3 → **4**
- 4.x = L9/10/11 (PHP 8.0+), default scaffolding is **Bootstrap 5** (was 4). Re-scaffold/reconcile auth views. Scaffolding-only — no runtime logic impact.

### laravel/tinker → keep `^2.7`. laravel/helpers → keep `^1` (ideally migrate code to `Str::`/`Arr::` and drop).

---

# 2. Spatie packages

> **Flysystem note:** no Spatie package declares flysystem directly. The 1→3 jump happens automatically via `illuminate/filesystem` when you go **L8→L9**. Audit custom disks/adapters at the L9 step (see flysystem-aws-s3-v3 below).

### spatie/laravel-medialibrary — HIGH RISK, do v7→v8→v9→v10 sequentially
Version map is **offset** from Laravel: L8 → v9, L9/L10 → **v10** (v10 dropped PHP 7; you do *not* need v11 for L10).

- **v7 → v8 (biggest structural break):**
  - DB migration: add `conversions_disk` (copy from `disk`), add `uuid` (backfill `Media::cursor()->each(fn($m)=>$m->update(['uuid'=>Str::uuid()]))`).
  - Trait rename `HasMediaTrait` → `InteractsWithMedia`; implement `HasMedia` interface.
  - **Namespaces changed for nearly every class** — audit all `use Spatie\MediaLibrary\...`.
  - Config rename `medialibrary.php` → `media-library.php`; commands `medialibrary:*` → `media-library:*` (update Kernel schedule).
  - Add `void` return type to `registerMediaConversions`/`registerMediaCollections`.
- **v8 → v9:** add `json` column `generated_conversions` (migrate values out of `custom_properties`); default collection `medialibrary_original` → `media_library_original` (rename generated files + `responsive_images` data); config `conversion_file_namer` → `file_namer`.
- **v9 → v10:** drops PHP 7 (needs PHP 8); flysystem 3 enters via L9; stays on `intervention/image ^2.7` (the Image-v3 rewrite is a v11 concern — defer by staying on v10).
- **Republish config at v8 and v9 (diff, don't overwrite). Run migrations at v8 + v9.**

### spatie/laravel-permission → **5**
- v5 spans `^8.12|^9|^10` (PHP 8.0+). v6 also covers L8–11.
- Republish `config/permission.php` (new keys: `cache.store`, teams, `display_permission_in_exception`); migration changed (nullable `team_id`) — reconcile, don't blindly re-run.

### spatie/laravel-query-builder → **5.7+** (mind the gap)
- 3.x/4.x = L6/7/8; **5.7.0** is the first with L9/10. v6 needs PHP 8.2 (pick if on 8.2).
- Breaking: **deny-by-default** — unspecified filters/sorts/includes now throw `InvalidFilterQuery`/`InvalidSortQuery` unless explicitly `allowed*()`. Tighten every QueryBuilder call site. Optional `config/query-builder.php`.

### spatie/laravel-schemaless-attributes → **2.4+**
- v2 prefers the **cast** approach (`'extra_attributes' => SchemalessAttributes::class`) over old accessor. No migration/config needed (column already exists).

### spatie/laravel-tags → **4**
- v3 changed `tags` table: `slug`/`name` became **JSON** (translations) + added `type`/`order_column` — needs the v3 migration to convert columns. Republish `tags.php`.

### spatie/laravel-webhook-client → **3**
- Single v3 covers L8–10 (PHP 8.0+). **Config restructured to a `configs` array** — republish/merge. `WebhookProfile`/`SignatureValidator`/`ProcessWebhookJob` signatures changed; register routes via `Route::webhooks('name')`. `webhook_calls` table migration updated.

### spatie/opening-hours (framework-agnostic) → v3 (PHP ≤8.1), **v4** (PHP 8.2)
- v4 needs PHP 8.2, native enums/readonly. Audit `Day` constants + exception types. No config/migrations.

### spatie/eloquent-sortable → **4** (⚠ needs PHP 8.0 — may force PHP 8 early)
- v4 = L8+ but **PHP 8.0/8.1 floor**. v5 = PHP 8.2/L10-13. `buildSortQuery()` added; config moved into model `$sortable` array.

### spatie/laravel-ray (dev) → keep `^1`. Already spans L7–L13. No action.

---

# 3. Nova third-party + charts

### titasgailius/search-relations → **`^2.0`** (actively maintained — v2 "Nova 5 Support" Feb 2026). Works Nova 4 & 5. v2 changed the trait/config API — re-check `$searchRelations`.

### dillingham/nova-attach-many → **`^2.0`** (Nova 4 only; **blocks future Nova 5**). Tagged 2.0.0 only allows L8/9 — if composer balks on L10, require `dev-master` (master allows `^10`).

### inspheric/nova-defaultable → **REMOVE, replace with Nova native** `->default(fn($request)=>...)`. Abandoned (incompatible with Nova ≥3.4). Low effort — mechanical find/replace of `->defaultable()` → `->default()`.

### koss-shtukert/...select2-auto-complete → **REMOVE, replace with Nova native** `BelongsTo::make(...)->searchable()->withSubtitles()` or `Select::make(...)->searchable()`. Dead since 2020; its Vue-2 assets won't run on Nova 4. Low–medium effort; ensure related resource defines `$search`.

### consoletvs/charts → **KEEP `^6`** (CORRECTED — it IS used)
- Usage audit found it active: `app/Charts/*` (Highcharts) rendered into **emailed/PDF reports** (`GetConversationsDetailedReport`, `SendWeeklyReportsToAccounts`) — **not** Nova dashboard cards, so Nova Metrics is the wrong replacement.
- v6 line still gets Laravel-compat bumps (6.9.0 = L12) and installs fine on L8/9/10 → **not a blocker; keep on `^6` through the upgrade.** Optional post-upgrade swap → Larapex Charts / maintained Highcharts wrapper. See [[Package Removability Audit]].

---

# 4. Auth / JWT / infrastructure

### tymon/jwt-auth → **`^2.0`** (after PHP 8)
- 1.0.2 = last 1.x, **no PHP 8 / no L9+**. 2.0.0 (Feb 2023) = L9/10, PHP 8.0+.
- Bumps `lcobucci/jwt` 3→4 (→5 at 2.3) — major rewrite (immutable tokens, `Configuration`/signer/key API). Update any custom lcobucci code.
- **Not redundant with Passport** — keep both (independent guards). ⚠ Align `lcobucci/jwt` between jwt-auth 2.x and Passport 11/12.
- Steps: bump, diff `config/jwt.php`, existing secret is fine.

### firebase/php-jwt → `^6.10` (PHP 7.4-safe) → **`^7.0`** (PHP 8)
- **6.0 breaking:** `JWT::decode($jwt, new Key($key,'RS256'))` — 2nd arg must be a `Key` object now (not raw string); `JWT::encode($payload,$key,'HS256')` — alg now required; `JWK::parseKeySet()` returns `array<string,Key>`.
- **7.0:** key-size validation (small keys throw), PHP 8.0+.
- Do as its own focused pass — search every `JWT::decode`/`JWT::encode` call site. Framework-agnostic.

### doctrine/dbal → stay `^2` through L8/9, **`^3` at L10** (do NOT jump to 4 — L10 is built on DBAL 3; L11 *removes* DBAL entirely)
- 2→3 breaking: `Comparator` no longer static (`$schemaManager->createComparator()`); `listTableDetails()`/type mappings changed; `registerCustomDoctrineType` removed from `Schema\Builder` → use `DB::registerDoctrineType()`. Mostly affects `->change()` migrations + custom type registration.

### guzzlehttp/guzzle 6.5.8 (pinned) → **`^7.5`** (required at L9; allowed at L8 — do early)
- 7 = PSR-7 immutable + HandlerStack middleware; no `$response->json()` (decode body yourself); pulls `guzzlehttp/psr7 ^2`. Audit direct Guzzle usage (Laravel `Http` facade unaffected).

### guzzlehttp/psr7 1.9 → **`^2`** (rides with guzzle)
- Message objects strict/immutable — `with*()` returns new instance (capture the return value; in-place mutation silently no-ops).

### league/flysystem-aws-s3-v3 ~1.0 → **`^3.0`** (required at L9)
- Install: `composer require -W league/flysystem-aws-s3-v3 "^3.0"` (the `-W` is required).
- Adapter renamed `AwsS3v3\AwsS3Adapter` → `AwsS3V3\AwsS3V3Adapter` (only if instantiated directly).
- **Behavioral (Storage facade): writes overwrite by default + return `false` instead of throwing** (set `'throw'=>true` in disk config to restore); missing-file read returns `null`; cached adapters removed (drop `cache` key); custom `Storage::extend()` must return `FilesystemAdapter`.

### fideloper/proxy → **REMOVE at L9** (built in)
1. `TrustProxies.php`: `use Illuminate\Http\Middleware\TrustProxies as Middleware;`
2. Replace `$headers = Request::HEADER_X_FORWARDED_ALL` with the OR'd `HEADER_X_FORWARDED_FOR|HOST|PORT|PROTO|AWS_ELB`.
3. `composer remove fideloper/proxy`. Keep `$proxies`.

### fruitcake/laravel-cors → **REMOVE at L9** (built in since L9.2)
1. `Kernel.php`: `\Fruitcake\Cors\HandleCors::class` → `\Illuminate\Http\Middleware\HandleCors::class`.
2. `composer remove fruitcake/laravel-cors`. Keep `config/cors.php` (same keys; `fruitcake/php-cors` comes transitively via framework).

### facade/ignition (dev) → **spatie/laravel-ignition** at L9
- `^1.0` at L9, **`^2.0` at L10** (PHP 8.1+). Coupled with collision (6→7). Drop-in; diff `ignition.php`/`flare.php` if published.

---

# 5. Misc feature + dev packages

### lorisleiva/laravel-actions — MEDIUM rewrite at L9 (installed: v0.3.0, old API)
Verified usage: 53 classes `extends Action`, 45 use `authorize()`/`rules()`, 32 use the attributes bag, 25 use `actingAs()`/`user()`, none used as controllers/jobs. 2.x removes all those features → forced rewrite. **Full per-class migration recipe + effort breakdown in [[Package Removability Audit]].** Staged path:
1. **L8/PHP7.4:** `0.3 → ^1.2.2` (same old API; tiny change; runs on PHP 7.4).
2. **L9:** `→ ^2.7` rewrite — `extends Action` → `use AsAction`; `(new X)->run()` → `X::run()`; add `WithAttributes` to keep the attributes bag; relocate `authorize()`/`rules()` to the calling FormRequest or an explicit `$this->validate()`; pass `$user` explicitly instead of `actingAs()`/`user()`.

### eduardokum/laravel-mail-auto-embed → **REMOVED in Phase 0** (unused)
- Was the *only* package hooking into Swiftmailer (it registered a Swift plugin to embed images — which is exactly why its 1.x broke on L9 and a 2.x rewrite existed). Since it's unused and dropped, that L9 blocker disappears entirely.

---

## Framework: Swiftmailer → Symfony Mailer (Laravel 9) — **near-zero impact for this app**

> Full deep dive (EOL/support dates, version mapping, config + DSN, syntax differences) in **[[Swiftmailer vs Symfony Mailer]]**.

**What changed:** Laravel 9 replaced the abandoned **Swiftmailer** with **Symfony Mailer** as the underlying transport library. This is flagged "High Impact" in the official L9 upgrade guide *in general*, because it breaks anything that touches the Swift layer directly.

**The general breaking surface (what usually needs work):**
- Raw `Swift_Message` / `Swift_Attachment` / `Swift_Image` construction.
- `Mailable::withSwiftMessage()` / `Message::getSwiftMessage()` callbacks → renamed to `withSymfonyMessage()` and the object is now a `Symfony\Component\Mime\Email`.
- Custom mail transports registered via `Mail::extend()` (must now return a Symfony `TransportInterface`).
- `MessageSent` event: `$event->message` is now a `Symfony\…\Email` (not `Swift_Message`); `swift()` helper gone.
- The `failures()` method on the mailer was removed.
- Attachment data API tweaks (`->attachData()` MIME-type handling).

**This codebase's actual exposure — audited, essentially nil:**
- ✅ **No `Swift_*` usage anywhere** (app/, config/).
- ✅ **No `withSwiftMessage()` / `getSwiftMessage()`** callbacks.
- ✅ **No custom `Mail::extend()` transports / TransportManager.**
- ✅ **No Mailables at all** — all mail goes out via **16 Notifications** using the `mail` channel + `MailMessage` builder (e.g. `AccountWeeklyReport`, `VerifyEmail`, `UserInvited`), which is a framework abstraction fully insulated from the transport swap.
- ✅ **No `MessageSent`/`swift()` listeners** touching the message object.
- ✅ `config/mail.php` is already the modern `mailers`-array format (the `MAIL_DRIVER`→`MAIL_MAILER` change was L6→L7, already done).

**Net: the Swift→Symfony switch requires ~no code changes here.** It's a dependency/config concern only. Post-upgrade verification checklist:
1. **Transport in use:** default is `smtp` (`config/services.php` also has `ses` + `mailgun` configured). **SMTP and SES need no extra package** on L9 (Laravel keeps its own `SesTransport` over `aws/aws-sdk-php`, which you already have).
2. **If Mailgun is actually used** (config present but default is smtp): L9 requires adding `symfony/mailgun-mailer` + `symfony/http-client`. Confirm whether any environment sets `MAIL_MAILER=mailgun`; if not, ignore.
3. Send one of each notification type on staging (verify, callback, weekly report with its Highcharts payload) and confirm delivery + formatting.
4. Re-check the markdown mail theme (`config/mail.php` `markdown.theme`) renders — no custom `resources/views/vendor/mail` overrides exist, so it uses the stock theme (safe).

### maatwebsite/excel → keep `^3.1` (bump to latest 3.1.x — covers L9/L10/PHP 8.2, no major breaks).
### mews/purifier → **`^3.4`** (no breaking changes from 3.3).
### chelout/laravel-relationship-events → `^1` for L8/L9, **`^2` at L10** (PHP 8.1+).
### staudenmeir/eloquent-json-relations → pin exact: **`1.7` (L9), `1.8` (L10)** — 1.x PHP floor crept up, don't use `^1.4`.
### kutia/larafirebase → `^1.3` (covers L9/L10) but **dormant** (no release since Nov 2023, no L11). Flag for future replacement (`kreait/laravel-firebase`).
### barryvdh/laravel-translation-manager → **`^0.6`** (needs PHP 8.1 — hold until L9+PHP8.1; keep 0.5.8 on 7.4).
### monicahq/laravel-cloudflare → **`^3`** (covers L8.54–12). Don't double-configure with built-in `TrustProxies` — it just supplies Cloudflare IP ranges. (L11 → 4.x + `bootstrap/app.php`.)
### torann/geoip → **`^3`** (after PHP 8). v3 = L8–13, PHP 8.0+.
### geoip2/geoip2 (MaxMind) → `^2` on PHP 7.4, **`^3` on PHP 8.1+** (torann/geoip v3 pulls a compatible version).
### aws/aws-sdk-php → **cap `<3.350` on PHP 7.4**, float after PHP 8.1. No API breaks — platform floor only.
### pusher/pusher-php-server 4 → **`^7.2`** (runs even on PHP 7.4 — upgrade early). Used for Sockudo via pusher protocol.
- **4→7 breaking:** HTTP client now Guzzle (hard dep); `host`/`port`/`timeout`/`curl_options` constructor params removed; `trigger()` 3rd arg now `$params` array (was `$socket_id`); methods **throw `ApiErrorException`** on non-2xx (was `false`); `$app_id` strictly string. Audit direct `Pusher` construction + `trigger()` in broadcasting/Sockudo glue (Laravel's broadcaster wraps most).

### Dev:
- **fzaninotto/faker → fakerphp/faker** (abandoned). `composer remove --dev fzaninotto/faker && composer require --dev fakerphp/faker`. `Faker\` namespace unchanged — drop-in. Do early.
- **nunomaduro/collision** → `^5` (L8) → **`^6` (L9/L10)**. Lockstep with Laravel major.
- **phpunit/phpunit** `^8.5` → **`^9.5`** (L8/L9). At L10 keep `^9.6` (lowest friction) or jump `^10.1` as a separate isolated step (PHPUnit 10 needs PHP 8.1, changed XML schema, remove `processUncoveredFiles`).
- **mockery/mockery** → **`^1.6`** across L8/9/10.
- **barryvdh/laravel-debugbar** → **`^3.9`**. **barryvdh/laravel-ide-helper** → **`^2.13`** (stay 2.x for ≤L10; 3.x only at L11).
- **beyondcode/laravel-dump-server** — *not* abandoned (v2.1 supports L12). Either bump `^2.1` or drop (`composer remove --dev`, delete published config). No urgency.

---

## Lockstep-with-Laravel-major packages (pin exactly per major, don't use `^`)
`nunomaduro/collision` (5→6) · `chelout/laravel-relationship-events` (1→2) · `staudenmeir/eloquent-json-relations` (1.7→1.8) · `barryvdh/laravel-ide-helper` (stay 2.x ≤L10) · `monicahq/laravel-cloudflare` (3.x) · `laravel/passport` · `laravel/nova`.

## Highest composer-resolution risk points
1. **tymon/jwt-auth 2.x + Passport 11/12 sharing `lcobucci/jwt`** — verify versions align.
2. **Flysystem 3 behavioral change** — writes return `false` instead of throwing.
3. **PHP-floor traps** during the 7.4 phase (see top table).

## Watch-later (fine for L10, breaks at L11)
`kutia/larafirebase` (dormant, no L11) · `laravel-ide-helper` (need 3.x) · `monicahq/laravel-cloudflare` (4.x + bootstrap/app.php) · `dillingham/nova-attach-many` (Nova-4-only, blocks Nova 5).
