# UI Internationalization (i18n) — Plan & Architecture

> **Scope:** translate the **operator dashboard app** — the web app where our
> users (operators/agents) log in — into multiple languages (menu, pages, labels,
> API messages). This is **not** the visitor-facing chat widget and **not** the KB;
> those have their own separate multi-language content systems (see §5).
>
> **Roles:** this repo is the **backend (API)**. Its job is to (1) resolve the
> effective language and (2) **send the translation data to the frontend app** that
> the operator's browser loads. The frontend is a separate SPA repo that just
> renders what the backend sends. The effective language is resolved with a 3-level
> precedence: **config default → account settings → user settings**, with an
> optional live per-request override.

---

## 1. Goals

- Show the UI (menu items, page titles, labels, buttons, validation/API messages)
  in the user's preferred language.
- Resolve the active locale from a clear precedence chain:
  1. **Config default** — `config('app.locale')` (`en`).
  2. **Account setting** — `account.settings.locale` (the workspace default a team admin picks).
  3. **User setting** — `user.settings.locale` (each agent's personal override).
  4. *(optional)* **Live request override** — `x-lc-locale` header, so a user can
     switch language in the current session without persisting.
- Most-specific wins: **user → account → config**.
- Support the four locales already configured: `en`, `ka`, `ru`, `he`
  (`he` is RTL — must be exposed to the UI).

## 2. What already exists (reuse, don't rebuild)

| Concern | Where | Notes |
|---|---|---|
| Locale config | [config/app.php:89-104](config/app.php) | `locale=en`, `locales=['en','ka','ru','he']`, `fallback_locale=en` |
| Per-request locale | [app/Http/Middleware/Localization.php](app/Http/Middleware/Localization.php) | **Global** middleware ([app/Http/Kernel.php:62](app/Http/Kernel.php)). Currently only reads `x-lc-locale` header. |
| Account locale accessor | [app/Models/Account.php:306-309](app/Models/Account.php) | `getLocale()` → `$this->setting('locale', app()->getLocale())` |
| Settings storage | [app/Models/Traits/HasSettings.php](app/Models/Traits/HasSettings.php) | JSON `settings` column on both `Account` and `User`; `->setting($key, $default)` getter, `->setting([k=>v])->save()` setter |
| Message translation files | `resources/lang/{en,ka,ru,he}/*.php` + `resources/lang/{ka,he}.json` | Laravel `__()` already used in controllers |
| Translation management UI | `barryvdh/laravel-translation-manager` (composer) | Can be used to manage strings |
| Language catalog | [app/Models/Language.php](app/Models/Language.php) + `GET /api/languages` | Full country/language reference list (separate from UI locales) |

**Key fact:** This repo is an **API backend** for a separate SPA dashboard
(`resources/views` is empty; Nova is the only server-rendered admin). So i18n has
**two surfaces**:

- **A. Server strings** — API success/error/validation messages returned in JSON,
  resolved with Laravel `__()` against `resources/lang/*`.
- **B. SPA chrome** — menu, page titles, buttons, labels rendered by the frontend.

**Decision (confirmed):** the API is the single source of truth for **both**. The
backend **sends the full translation bundle to the frontend at load time** (along
with the resolved locale), so the SPA does not ship its own string files. The
frontend just renders keys against the bundle it receives. This is "Strategy B"
below; Strategy A is kept only for historical context.

This plan covers the **API side**. A short companion section describes the
frontend contract.

---

## 3. Architecture

### 3.1 Locale resolution (the core change)

Centralize precedence in one resolver, called by the existing global middleware.

```
effectiveLocale =
       header  x-lc-locale            (if present AND in app.locales)   // live override
    ?? user.settings.locale           (if authenticated)                // personal
    ?? account.settings.locale        (if authenticated)                // workspace
    ?? config('app.locale')                                             // default
```

> Note on ordering: the **header** sits on top only as a transient override (the
> SPA sends it when the user flips a language switcher before it's persisted). The
> *persisted* precedence the user asked for is **config → account → user**, i.e.
> user overrides account overrides config. If you don't want a live override at
> all, drop the header branch and persist the user's choice to `user.settings.locale`.

A small `LocaleResolver` service keeps this testable and reusable (jobs, queued
notifications, broadcasting all need it too — not just HTTP).

```php
// app/Services/Localization/LocaleResolver.php
namespace App\Services\Localization;

use App\Models\Account;
use App\Models\User;

class LocaleResolver
{
    public function resolve(?User $user, ?string $headerLocale = null): string
    {
        $supported = config('app.locales', []);

        if ($headerLocale && in_array($headerLocale, $supported, true)) {
            return $headerLocale;
        }

        if ($user) {
            $userLocale = $user->setting('locale');
            if ($userLocale && in_array($userLocale, $supported, true)) {
                return $userLocale;
            }

            $account = $user->account; // adjust to the real relationship name
            if ($account) {
                $accountLocale = $account->setting('locale');
                if ($accountLocale && in_array($accountLocale, $supported, true)) {
                    return $accountLocale;
                }
            }
        }

        return config('app.locale');
    }
}
```

### 3.2 Update the global `Localization` middleware

It already runs globally and early. Extend it to use the resolver. Because the
middleware is in the **global** stack (before route auth), `auth()->user()` may
not be resolved yet for token-guarded routes; resolve the user via the configured
API guard explicitly, and fall back gracefully.

```php
public function handle($request, Closure $next)
{
    $user = $request->user(); // works once Passport has authenticated the bearer token
    $header = $request->header('x-lc-locale');

    app()->setLocale($this->resolver->resolve($user, $header));

    return $next($request);
}
```

> ⚠️ **Auth timing caveat.** In the global stack, `$request->user()` can be `null`
> for stateless token auth because auth middleware hasn't run yet. Two options:
> 1. **Move locale resolution into the `auth:api` route-middleware group** (or a
>    new `SetUserLocale` middleware appended after auth) so the user is available.
>    Keep the current header-only `Localization` global as a pre-auth default.
> 2. Resolve the token manually in the global middleware
>    (`Auth::guard('api')->user()`). Simpler to keep one middleware, but couples
>    it to the guard.
>
> **Recommended:** keep global `Localization` for header/config default, and add a
> lightweight `SetUserLocale` middleware to the authenticated route group that
> upgrades the locale to account/user once the user is known.

### 3.3 Persisting preferences (no schema changes needed)

Both `Account` and `User` already have a JSON `settings` column via `HasSettings`.
Store the choice under the `locale` key — `Account::getLocale()` already reads it.

- **Account default** — set by an account admin:
  ```php
  $account->setting(['locale' => $validated['locale']])->save();
  ```
- **User override** — set by each agent:
  ```php
  $request->user()->setting(['locale' => $validated['locale']])->save();
  ```

Add (or extend) endpoints:

| Method | Route | Action | Auth |
|---|---|---|---|
| `PATCH` | `/api/account/settings` (or existing account update) | set `settings.locale` | account admin |
| `PATCH` | `/api/user/settings` or `/api/profile` (existing `UserController`) | set `settings.locale` | self |
| `GET` | `/api/locales` | list supported UI locales + metadata (label, native name, `dir`) | any |

Validation rule for both setters: `'locale' => ['required', Rule::in(config('app.locales'))]`.

> Keep the existing `GET /api/languages` (the big country list) separate from
> `GET /api/locales` (the 4 UI languages). They serve different purposes.

### 3.4 Delivering locale + the full translation bundle on load

**The frontend receives everything it needs to render the UI in one shot when it
loads.** Attach the resolved locale, its metadata, the list of available locales,
**and the complete translation bundle** for the effective locale to the bootstrap/
profile endpoint the SPA already calls on startup (e.g. the `me`/profile resource).
This avoids a separate round-trip before the menu can render.

```json
{
  "locale": "ka",
  "fallback_locale": "en",
  "dir": "rtl_or_ltr",
  "available_locales": [
    { "code": "en", "label": "English",  "native": "English",  "dir": "ltr" },
    { "code": "ka", "label": "Georgian", "native": "ქართული", "dir": "ltr" },
    { "code": "ru", "label": "Russian",  "native": "Русский",  "dir": "ltr" },
    { "code": "he", "label": "Hebrew",   "native": "עברית",    "dir": "rtl" }
  ],
  "translations": {
    "menu.dashboard": "დაფა",
    "menu.conversations": "საუბრები",
    "menu.contacts": "კონტაქტები",
    "menu.settings": "პარამეტრები",
    "pages.settings.title": "ანგარიშის პარამეტრები",
    "buttons.save": "შენახვა"
    // … the full key→value map for the resolved locale
  }
}
```

The `translations` map is the merged result of the locale's strings layered on top
of the fallback (`en`), so every key is always present even if a translation is
missing (see §3.6 for keying and merge logic).

### 3.5 Also expose a standalone bundle endpoint (for language switching)

So the user can switch language **without a full reload**, expose the same bundle
on its own. The frontend fetches the new locale's bundle on switch (and sends
`x-lc-locale` / PATCHes the user setting):

```php
// routes/api.php
Route::get('translations/{locale}', [TranslationController::class, 'show'])
    ->name('translations.show');
```

```php
// app/Http/Controllers/TranslationController.php
public function show(string $locale)
{
    abort_unless(in_array($locale, config('app.locales'), true), 404);

    $strings = app(TranslationBundle::class)->for($locale); // merged over fallback

    return response()->json([
        'locale'      => $locale,
        'dir'         => config("app.locale_dirs.$locale", 'ltr'),
        'translations'=> $strings,
    ])->setMaxAge(86400)->setPublic(); // cache; bust on deploy via version/ETag
}
```

The bootstrap response (§3.4) and this endpoint share the **same**
`TranslationBundle` service so there is one source of truth.

### 3.6 The `TranslationBundle` service (single source of truth)

Builds the flat key→value map the frontend consumes, merging the requested locale
over the fallback so no key is ever missing.

```php
// app/Services/Localization/TranslationBundle.php
namespace App\Services\Localization;

use Illuminate\Support\Facades\Cache;

class TranslationBundle
{
    public function for(string $locale): array
    {
        return Cache::rememberForever("ui_translations.$locale", function () use ($locale) {
            $fallback = $this->load(config('app.fallback_locale'));
            $current  = $this->load($locale);
            return array_merge($fallback, $current); // current overrides fallback
        });
    }

    private function load(string $locale): array
    {
        // Source of UI strings. Recommended: a dedicated namespace/file so UI copy
        // is not tangled with auth/validation .php files.
        //   resources/lang/{locale}.json            → general strings, OR
        //   resources/lang/{locale}/ui.php          → namespaced ('ui.menu.dashboard')
        $path = resource_path("lang/{$locale}.json"); // Laravel 7
        return is_file($path) ? json_decode(file_get_contents($path), true) ?? [] : [];
    }
}
```

> **Cache busting:** include a `translations_version` (e.g. a deploy hash or
> `filemtime`) in the bootstrap payload and/or as a query param on
> `GET /api/translations/{locale}` so the SPA knows when to refetch and HTTP
> caches invalidate. Clear `ui_translations.*` cache keys on deploy.

> **Where UI strings live:** keep menu/page/label keys in their **own** file
> (e.g. `resources/lang/{locale}.json` or a `ui.php` namespace) — separate from
> Laravel's `auth.php`/`validation.php`. Non-developers can edit these via
> `laravel-translation-manager`.

---

## 4. RTL support

`he` is right-to-left. Expose `dir` (`ltr`/`rtl`) per locale (3.4) so the SPA can
set `<html dir>` and mirror layout. Maintain an explicit map:
`['en'=>'ltr','ka'=>'ltr','ru'=>'ltr','he'=>'rtl']` (e.g. in
`config/app.php` as `locale_dirs` or in the `LocaleResolver`/`/api/locales`
resource).

## 5. Scope boundary — operator app UI vs. visitor-facing content

This plan covers **only the operator dashboard app's static UI strings** (menu,
pages, labels, buttons, system/validation messages) shown to logged-in
operators/agents.

It is **explicitly NOT** about the visitor-facing systems, which are seen by the
end customer (not the operator) and already have their own separate multi-language
content patterns. Do **not** merge these into the operator UI locale system:

- **Chat widget** — `Widget.translations` JSON + `PUT /api/widgets/{id}/translations`
  (the embeddable widget visitors interact with on customer sites).
- **KB / KbArticle** — per-language content (`language` column, JSON fields), the
  public help center visitors read.
- **Triggers / canned messages** — locale-keyed bodies sent to visitors.

These each resolve their own language from the **visitor's** context (widget
config, KB URL, conversation), which is independent of the **operator's** UI
locale that this plan resolves from config → account → user.

---

## 6. Implementation steps

1. **Add `config('app.locale_dirs')`** (RTL map) and confirm `app.locales`.
2. **Create `App\Services\Localization\LocaleResolver`** (precedence logic, §3.1) + unit tests.
3. **Refactor `Localization` middleware** to header+config only (pre-auth default).
4. **Add `SetUserLocale` middleware** to the authenticated route group; it calls
   the resolver with the now-known user and upgrades the app locale.
5. **Add `GET /api/locales`** + `LocaleResource` (code, label, native, dir).
6. **Add locale setters:** extend account update + user/profile update to accept
   and validate `settings.locale`.
7. **Create `TranslationBundle` service** (§3.6): load + merge-over-fallback +
   cache the flat key→value UI string map. Define where UI strings live and
   migrate menu/page/label copy into that file/namespace.
8. **Attach `locale`/`dir`/`available_locales`/`translations`/`translations_version`
   to the bootstrap/profile (`me`) response** (§3.4) — this is what makes the menu
   render translated on first load.
9. **Add `GET /api/translations/{locale}`** (§3.5) sharing `TranslationBundle`, for
   language switching without a full reload. Cache + version it.
10. **Audit API strings:** ensure user-facing controller/validation messages use
    `__()` / translation keys; fill `resources/lang/{ka,ru,he}` gaps.
11. **Frontend (separate repo):** read `translations` from bootstrap to render the
    menu/UI, set `<html lang dir>`, add a language switcher that PATCHes user
    settings, sends `x-lc-locale`, and refetches `GET /api/translations/{locale}`.
12. **Tests:** resolver precedence (config-only, account-only, user-over-account,
    header-override, invalid/unsupported locale → fallback); `TranslationBundle`
    merge (missing key falls back to `en`); bootstrap payload includes a complete
    bundle; middleware sets `app()->getLocale()` correctly per authenticated user.

## 7. Edge cases & decisions

- **Unsupported/invalid locale** anywhere in the chain → skip that level, fall
  through to the next; ultimately `config('app.locale')`.
- **Missing translation key** → Laravel falls back to `fallback_locale` (`en`),
  then the key itself. Ensure `en` is complete.
- **Queued jobs / notifications / emails** must not rely on `app()->getLocale()`
  set by HTTP middleware. Pass the target user's locale explicitly (use
  `LocaleResolver` + `App::setLocale()` inside the job, or Laravel's
  `Notification` locale).
- **Caching:** `GET /api/locales` and translation bundles are cache-friendly;
  bust on deploy / locale config change.
- **Auth timing** (§3.2) is the main implementation risk — validate where
  `$request->user()` becomes available in the middleware pipeline.

## 8. Open questions for the team

1. Confirm the SPA bootstrap/`me` endpoint name to attach `locale` + `translations` to.
2. ~~Strategy A vs. B~~ — **Decided: API sends the full bundle on load (Strategy B).**
3. Where should UI strings live — a single `resources/lang/{locale}.json`, or a
   `ui.php` namespace? (Affects key naming, e.g. `menu.dashboard` vs `ui.menu.dashboard`.)
4. Should a live `x-lc-locale` override exist, or only persisted user/account
   settings? (Affects whether the header stays at the top of precedence.)
5. Confirm the `User → Account` relationship name used in `LocaleResolver`.
6. Any locales to add beyond `en/ka/ru/he`?
