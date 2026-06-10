# PHP 7.4 → 8.x & Laravel Upgrade Plan

> LiveCaller API backend. Context: Laravel WebSockets code has been removed and **Sockudo** is the working broadcasting solution in production. This plan covers the gradual PHP 7.4 → 8.x upgrade with the Laravel + package upgrades that must move alongside it.

---

## Current State (verified from the codebase)

| Item | Now | Notes |
|---|---|---|
| PHP | **7.4** (Docker: Debian + sury.org `php7.4-*`) | Host has 8.4, but containers run 7.4 |
| Laravel | **7.30.4** | `^7.0` |
| **Laravel Nova** | **3.0.x-dev** | 14 resources, 29 custom files, `nova-components/` — **the hard blocker** |
| Cashier 12.7 · Horizon 4.3 · Passport 9.3 · Telescope 3.5 · UI 2.5 | L7-era | each tied to framework major |
| Dead: `beyondcode/laravel-websockets ^1.4` | still required | + `config/websockets.php`, `Middleware/Websockets/Authorize.php`, plus compiled `event.so` in Docker — all orphaned by the Sockudo move |
| Old: jwt-auth 1.0.2, guzzle 6.5.8 (pinned), doctrine/dbal ^2, faker = `fzaninotto` (abandoned), `fideloper/proxy`, `fruitcake/laravel-cors`, `facade/ignition`, `consoletvs/charts` | | several become built-in or get renamed in later Laravel |

Broadcasting already runs through the **`pusher` driver → host `127.0.0.1:6001`** (Sockudo). So the websockets package/config/middleware are confirmed dead weight.

---

## The governing constraint: compatibility matrix

You cannot skip Laravel majors — each one pulls its own ecosystem. And **Nova gates everything**, because Nova majors are paid and require resource-API rework.

| Laravel | PHP range | Nova that supports it |
|---|---|---|
| 7 | 7.2–8.0 | Nova 3 |
| 8 | 7.3–8.1 | Nova 3 **or** 4 |
| 9 | **8.0**–8.2 | Nova 4 |
| 10 | **8.1**–8.3 | Nova 4 **or** 5 |
| 11 | **8.2**–8.4 | Nova 4 **or** 5 |

**Key insight:** Nova 3 dies at Laravel 8. The moment you go to Laravel 9 you **must** be on Nova 4 (paid license + resource changes). So the natural "free" milestone is **Laravel 8 / Nova 3**, and the first "costs money + real work" jump is **Laravel 9 / Nova 4**.

**Recommended target:** land on **Laravel 10 + PHP 8.2 + Nova 4** as the stable stopping point (all in active support), with Laravel 11 as an optional follow-on. Don't try to reach 11 in one campaign.

---

## Phase 0 — Prep & dead-code removal (stay on L7 / PHP 7.4)

Do this *first*, on its own PR, so the later upgrades start from a clean base. **All removals below are codebase-verified zero-usage — see [[Package Removability Audit]].**

1. **Remove the dead websocket stack:**
   - `composer remove beyondcode/laravel-websockets`
   - Delete `config/websockets.php`, `app/Http/Middleware/Websockets/Authorize.php`, and the websocket route registration in `app/Providers/BroadcastServiceProvider.php`.
   - Remove the **`event.so`** build steps + `event.ini` from `docker-containers/php-api/Dockerfile` and `docker-php/php-api/Dockerfile` (it existed only for laravel-websockets' async server). Verify Sockudo broadcasting still works end-to-end after.
2. **Drop unused production packages (verified no code usage):**
   - `composer remove laravel/ui` — no `Auth::routes()`/scaffolding/views (API backend).
   - `composer remove barryvdh/laravel-translation-manager` — only a dead config file; delete `config/translation-manager.php`.
   - `composer remove eduardokum/laravel-mail-auto-embed` — no mailables/`@embed`. **Bonus: skips a full 1→2 Symfony-Mailer rewrite at L9.**
   - `composer remove inspheric/nova-defaultable` — used in zero Nova resources.
   - `composer remove kutia-software-company/larafirebase` — no FCM/push usage.
3. **Drop unused dev tools:**
   - `composer remove --dev beyondcode/laravel-dump-server` (superseded; unused).
   - **Keep `spatie/laravel-ray`** (team uses Ray) — just bump within `^1`. Keep `barryvdh/laravel-debugbar` + `barryvdh/laravel-ide-helper` if devs use them locally (bump per per-package doc), else drop.
4. **Swap abandoned dev dep:** `fzaninotto/faker` → `fakerphp/faker` (`composer remove --dev fzaninotto/faker && composer require --dev fakerphp/faker`; `Faker\` namespace unchanged). Required anyway for L8.
5. **Establish a safety net:** make sure the test suite runs green in CI against the 7.4 container, and stand up a **staging environment** mirroring prod. Every later phase deploys to staging first. Without this, the Nova/medialibrary jumps are very risky.
6. **Pin a baseline:** tag the repo (`pre-upgrade-baseline`) so you can roll back.

> No version bumps yet — this phase only removes weight and de-risks. Ship and run it in prod for a few days.
>
> **Optional extra cleanup** (sheds two more upgrade targets): migrate off `spatie/laravel-tags` (only its `HasSlug` trait is used — the tagging itself is a custom `app/Models/Traits/HasTags.php`; replace `HasSlug` with `Str::slug()` at ~4–5 sites) and `laravel/helpers` (~5 `array_first()`/`str_is()` sites → `Arr::first()`/`Str::is()`).

---

## Phase 1 — Laravel 7 → 8 (Nova stays 3; PHP stays 7.4)

The **cheapest** big step — Nova 3 supports L8, so no Nova cost.

**`composer.json` changes:**
```
"php": "^7.4|^8.0",
"laravel/framework": "^8.0",
// laravel/ui — REMOVED in Phase 0 (unused); skip this bump
"laravel/cashier": "^13.0",
"laravel/horizon": "^5.0",
"laravel/passport": "^10.0",
"laravel/telescope": "^4.0",
"guzzlehttp/guzzle": "^7.2",
"nunomaduro/collision": "^5.0",
"phpunit/phpunit": "^9.3",
"facade/ignition": "^2.5",        // still fine on L8
"doctrine/dbal": "^2.12",          // keep 2.x for now
```
**Code work:**
- **Model factories** → class-based factories + `HasFactory` (verify each file in `database/factories/`).
- Update `composer.json` `classmap` for seeders/factories as needed.
- `$casts` / `Schema::defaultStringLength` and other L8 upgrade-guide items.
- Bump spatie packages to their L8-compatible majors (medialibrary, permission, query-builder, tags, etc. — let `composer update` resolve).

Run `composer update`, fix failures, full regression on staging, ship.

---

## Phase 2 — PHP 7.4 → 8.0/8.1 (still Laravel 8)

Decouple the runtime bump from the framework bump so you isolate failures.

1. Update **both Dockerfiles** `php7.4-*` → `php8.1-*` (8.1 is the sweet spot: L8 supports it and L9/10 will need it).
2. Remove the stale `platform` overrides in `composer.json` (`ext-pcntl 7.2.2`, `ext-posix 7.2.2`, `ext-imagick 7.2.2`) — set `config.platform.php` to `8.1` instead so Composer resolves correctly.
3. Run the app under PHP 8.0/8.1 — watch for: `required → nullable` param order, `{}` string/array access removal, stricter type juggling. Run a static pass (`phpstan` or Rector `PHP_80`/`PHP_81` sets) to catch most.
4. Regression on staging, ship.

After this phase you're on **Laravel 8 + PHP 8.1 + Nova 3** — a clean platform with zero licensing spend, and the hardest "free" work is done.

---

## Phase 3 — Laravel 8 → 9 + Nova 3 → 4 (PHP 8.1) — the big one

This is where money and real effort land. Budget the most time here.

**Framework/ecosystem:**
```
"laravel/framework": "^9.0",
"laravel/nova": "~4.0",            // PAID upgrade — license + resource API changes
"laravel/cashier": "^14.0",
"laravel/passport": "^11.0",
"laravel/horizon": "^5.7",
"laravel/telescope": "^4.9",
"doctrine/dbal": "^3.0",
"spatie/laravel-medialibrary": "^10.0",   // major jump (flysystem 1→3)
```
**Removals (now built into Laravel 9):**
- `composer remove fideloper/proxy` → use framework's `TrustProxies`.
- `composer remove fruitcake/laravel-cors` → use framework's built-in CORS + `config/cors.php`.
- `facade/ignition` → **`spatie/laravel-ignition`** (package renamed/rewritten for L9).

**Heavy items:**
- **Nova 4:** new resource definitions, `make:` signatures, the 29 custom Nova files + `nova-components/` packages must each be upgraded to Nova 4 (Vue 3 / new field API). Single biggest sub-task — plan to touch every file under `app/Nova` and `nova-components/`.
- **Flysystem 1 → 3** via medialibrary 10 — S3 adapter config changes (you use `league/flysystem-aws-s3-v3`); verify all media conversions/uploads.
- **Symfony 5 → 6** under the hood. **Swiftmailer → Symfony Mailer: near-zero impact here** — the app sends mail only via 16 Notifications/`MailMessage` (no Mailables, no `Swift_*`, no `withSwiftMessage()`, no custom transports), so no code changes needed; just verify SMTP/SES delivery on staging (add `symfony/mailgun-mailer` only *if* Mailgun is actually used). Full audit + checklist in [[Package Upgrade Reference (per-package)]]; deep dive (EOL dates, config, syntax) in [[Swiftmailer vs Symfony Mailer]]. `eduardokum/laravel-mail-auto-embed` (the only Swift-coupled package) was already removed in Phase 0.
- Audit the remaining packages here:
  - `consoletvs/charts` — **KEEP on `^6`** (it IS used — Highcharts in emailed reports, not Nova cards; v6 installs fine on L9/L10). Not a blocker; revisit post-upgrade if desired. See [[Package Removability Audit]].
  - `lorisleiva/laravel-actions` — **MEDIUM effort, forced rewrite here.** 53 classes on the 0.x base-class API leaning on `authorize()`/`rules()`/attributes-bag/`user()`/`actingAs()` — all removed in 2.x. Stage `0.3 → ^1.2.2` at L8 (same API), then rewrite to `^2.7` here via `AsAction` + `WithAttributes`, relocating validation/auth. **3rd-biggest code item after Nova 4 + medialibrary** — see the deep-dive in [[Package Removability Audit]].
  - `tymon/jwt-auth` → `^2.0` (needed for the `widget` guard — Visitor JWT; **keep**, it's not redundant with Passport). ⚠ Align `lcobucci/jwt` with Passport 11.

Full regression — especially the **admin panel (Nova)** and **media uploads**. Ship behind staging sign-off.

---

## Phase 4 — Laravel 9 → 10 (PHP 8.1 → 8.2)

Comparatively light, since the ecosystem is now modern.

```
"php": "^8.1|^8.2",
"laravel/framework": "^10.0",
"laravel/nova": "~4.27",          // Nova 4 supports L10 — no new license
"laravel/cashier": "^15.0",
"laravel/passport": "^11.8",
"laravel/horizon": "^5.21",
"laravel/telescope": "^4.17",
"doctrine/dbal": "^3.5",
"phpunit/phpunit": "^10.0",
"nunomaduro/collision": "^7.0",
```
- Bump Dockerfiles `php8.1-*` → `php8.2-*`.
- Minimal code: native type hints on `User` model methods, `Str`/`Stringable` deprecations, predis/redis config check.
- Nova 4 keeps working — **no Nova relicense needed** for L10. (Nova 5 is optional, only required if you later pursue Laravel 11/12.)

**Stop here** for the stable target: **Laravel 10 + PHP 8.2 + Nova 4.**

---

## Phase 5 (optional, later) — Laravel 11 + PHP 8.3 + Nova 5

Only if you want the newest. Requires **Nova 4 → 5** (another paid jump, PHP 8.1+ min), PHP 8.3 containers, and adopting (optionally) the slimmed L11 app skeleton. Defer until 10/8.2 has been stable in prod.

---

## Package triage summary

| Package                                                | Fate                                                                                     |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| `beyondcode/laravel-websockets`                        | **Remove now** (Phase 0)                                                                 |
| `fzaninotto/faker`                                     | **Replace** → `fakerphp/faker` (Phase 0)                                                 |
| `fideloper/proxy`, `fruitcake/laravel-cors`            | **Remove** at L9 (built-in)                                                              |
| `facade/ignition`                                      | **Replace** → `spatie/laravel-ignition` at L9                                            |
| `consoletvs/charts`                                    | **Replace/drop** (abandoned) at L9                                                       |
| `tymon/jwt-auth 1.0.2`                                 | **Re-evaluate** — upgrade to ^2 or remove if Passport covers it                          |
| `lorisleiva/laravel-actions ^0.3`                      | **Fix constraint** → 2.x supporting target Laravel                                       |
| `doctrine/dbal ^2`                                     | ^2 → ^3 at L9                                                                            |
| `guzzle 6.5.8` (pinned)                                | → ^7.2 at L8                                                                             |
| `laravel/{cashier,horizon,passport,telescope,ui,nova}` | bump in lockstep each phase (tables above)                                               |
| spatie/*                                               | bump as `composer update` resolves per phase; medialibrary 7→10 is the notable one at L9 |

## Cross-cutting rules

- **One major per PR**, deploy to staging, soak, then prod. Never combine a Laravel bump and a PHP bump in the same PR (Phase 1 vs 2, Phase 3 vs 4 are deliberately split).
- Use **Rector** (Laravel + PHP level sets) and **PHPStan** to mechanize each step and catch regressions.
- The two **biggest risk areas** are **Nova 4** (Phase 3) and **medialibrary 10 / flysystem 3** (Phase 3) — give them the most test attention.
- Run all `composer`/`artisan` inside the `livecaller-php-api-1` container, and update **both** `docker-containers/php-api/Dockerfile` and `docker-php/php-api/Dockerfile` at each PHP bump.

## Milestone summary (quick reference)

| Phase | Laravel | PHP | Nova | Cost / Risk |
|---|---|---|---|---|
| 0 | 7 | 7.4 | 3 | Free / low — cleanup |
| 1 | 8 | 7.4 | 3 | Free / medium — factories |
| 2 | 8 | 8.1 | 3 | Free / medium — runtime |
| 3 | 9 | 8.1 | **4** | **Paid / high** — Nova + medialibrary |
| 4 | 10 | 8.2 | 4 | Free / low |
| 5 (opt) | 11 | 8.3 | **5** | Paid / medium |
