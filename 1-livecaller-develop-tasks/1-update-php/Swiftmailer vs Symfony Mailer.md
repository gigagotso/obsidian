# Swiftmailer vs Symfony Mailer — deep dive

> Reference for the Laravel 9 mail-transport swap, part of **[[PHP 7.4 to 8.x & Laravel Upgrade Plan]]**. Covers EOL/support dates, version mapping, configuration, and syntax differences. (This app's actual exposure is near-zero — see the callout at the end and [[Package Upgrade Reference (per-package)]].)

---

## 1. What & why

**Swiftmailer** was the standalone PHP mailing library Laravel used under the hood through **Laravel 8**. Its codebase dated back to a v4.0 architecture (≈Symfony 1.2 era), had a single maintainer, and could not keep pace with modern PHP.

**Symfony Mailer** is its official successor — same concepts, modern PHP, maintained by the whole Symfony core team, with built-in DKIM/S-MIME signing, Twig integration, and first-class third-party provider bridges. It was introduced in **Symfony 4.3 (May 2019)** and reached feature parity with Swiftmailer in **Symfony 5.3**. Laravel switched to it in **Laravel 9.0 (Feb 2022)**.

The `swiftmailer/swiftmailer` Packagist package is now marked **abandoned**, explicitly suggesting `symfony/mailer` as the replacement.

---

## 2. End-of-life & support dates

### Swiftmailer
| Fact | Value |
|---|---|
| Last release | **6.3.0** (Oct 2021) |
| **End of maintenance (EOL)** | **30 November 2021** (coincided with Symfony 5.4 LTS / 6.0) |
| Status now | **Abandoned** — no security fixes since EOL |
| PHP support | 7.0 – 8.1 (6.3.0 added 8.1). **No official PHP 8.2+** — a hard blocker for modern PHP |

> Because Swiftmailer caps out at PHP 8.1, staying on it would itself block the PHP 8.2 target — independent of Laravel. The L9 switch removes that wall.

### Symfony Mailer (`symfony/mailer`)
Follows the standard Symfony release calendar: **standard releases = 8 months** of fixes; **LTS (the `.4` of each major) = 3 years bug fixes + 1 extra year security (4 yrs total)**.

| Version | Released | Active support ends | Security (EOL) | LTS | PHP floor |
|---|---|---|---|---|---|
| **5.4** | 29 Nov 2021 | 30 Nov 2024 | **28 Feb 2029** | ✅ | 7.2.5+ |
| 6.0 | 29 Nov 2021 | 31 Jan 2023 | 31 Jan 2023 | — | 8.0.2+ |
| 6.2 | 30 Nov 2022 | 31 Jul 2023 | 31 Jul 2023 | — | 8.1+ |
| **6.4** | 29 Nov 2023 | 30 Nov 2026 | **30 Nov 2027** | ✅ | 8.1+ |
| 7.0 | 29 Nov 2023 | 31 Jul 2024 | 31 Jul 2024 | — | 8.2+ |
| 7.2 | 29 Nov 2024 | 31 Jul 2025 | 31 Jul 2025 | — | 8.2+ |
| **7.4** | 27 Nov 2025 | 30 Nov 2028 | **30 Nov 2029** | ✅ | 8.2+ |

**Implication for this upgrade:** landing on **Laravel 10 pulls `symfony/mailer ^6.2`**, but Composer will resolve to the latest 6.x (6.4 LTS), which is supported through **Nov 2027**. Good runway. (Laravel 11/12 would move you to `^7`, with 7.4 LTS supported to **Nov 2029**.)

---

## 3. Version mapping (Laravel ↔ mail library ↔ PHP)

| Laravel | Mail library under the hood | Pulled version | PHP |
|---|---|---|---|
| ≤ 8 | **Swiftmailer** | `swiftmailer/swiftmailer ^6.0` | 7.2 – 8.1 |
| **9** | **Symfony Mailer** | `symfony/mailer ^6.0` | 8.0.2+ |
| **10** | Symfony Mailer | `symfony/mailer ^6.2` (resolves to 6.4 LTS) | 8.1+ |
| 11 | Symfony Mailer | `symfony/mailer ^7.0` | 8.2+ |
| 12 | Symfony Mailer | `symfony/mailer ^7.0` (→ 7.4 LTS) | 8.2+ |

You do **not** require `symfony/mailer` directly — `laravel/framework` pulls it transitively. Bumping the framework is what swaps the library.

---

## 4. Configuration differences

### `config/mail.php`
The **structure is essentially unchanged** between the two eras — the modern `'mailers' => [...]` array predates the switch (it arrived in Laravel 7, replacing the old single `MAIL_DRIVER` block). So no config-file restructuring is needed going L8→L9. The differences are in transport semantics and a few new keys.

```php
// Both eras — same shape:
'default' => env('MAIL_MAILER', 'smtp'),
'mailers' => [
    'smtp' => [
        'transport'  => 'smtp',
        'host'       => env('MAIL_HOST'),
        'port'       => env('MAIL_PORT', 587),
        'encryption' => env('MAIL_ENCRYPTION', 'tls'),
        'username'   => env('MAIL_USERNAME'),
        'password'   => env('MAIL_PASSWORD'),
    ],
    'ses' => ['transport' => 'ses'],
    'mailgun' => ['transport' => 'mailgun'],
    'sendmail' => ['transport' => 'sendmail', 'path' => '/usr/sbin/sendmail -bs'],
    'log' => ['transport' => 'log'],
    'array' => ['transport' => 'array'],
],
```

### Key behavioural/config differences

| Aspect | Swiftmailer (≤L8) | Symfony Mailer (L9+) |
|---|---|---|
| **SMTP encryption** | `encryption` = `tls`/`ssl` explicitly | Still reads `encryption`, but Symfony **auto-negotiates**: STARTTLS on 587, implicit TLS (smtps) on 465. A new `MAIL_SCHEME` / `scheme` key can be set explicitly. |
| **DSN support** | none | Optional `'url' => env('MAIL_URL')` per mailer — a single Symfony **DSN** string replaces host/port/user/pass, e.g. `smtp://user:pass@host:587`. |
| **`local_domain`** | n/a | New optional `local_domain` key (HELO/EHLO name). |
| **Failover/round-robin** | manual | Native: `'transport' => 'failover'` / `'roundrobin'` with a `mailers` list. |
| **Sendmail path** | `path` | `path` (now full command incl. `-bs`). |

### Symfony Mailer DSN scheme cheat-sheet (if you adopt `MAIL_URL`)
```
smtp://user:pass@smtp.example.com:587      # SMTP (STARTTLS)
smtps://user:pass@smtp.example.com:465     # SMTP (implicit TLS)
sendmail://default
ses+api://ACCESS_KEY:SECRET@default        # Amazon SES (API)
ses+smtp://...                             # Amazon SES (SMTP)
mailgun+https://KEY:DOMAIN@default
postmark+api://TOKEN@default
```

### Third-party transport packages — the real practical change
With Swiftmailer, Laravel bundled the drivers. With Symfony Mailer, **each provider is a separate Symfony "bridge" package** you must `composer require` if you use it:

| Provider                          | Swiftmailer era   | Symfony Mailer era (L9+)                                                                   |
| --------------------------------- | ----------------- | ------------------------------------------------------------------------------------------ |
| **SMTP / sendmail / log / array** | built-in          | built-in (no extra package)                                                                |
| **Amazon SES**                    | built-in (Guzzle) | **Laravel keeps its own `SesTransport` over `aws/aws-sdk-php`** — no Symfony bridge needed |
| **Mailgun**                       | built-in          | **`symfony/mailgun-mailer` + `symfony/http-client`**                                       |
| **Postmark**                      | built-in          | **`symfony/postmark-mailer` + `symfony/http-client`**                                      |
| **Resend / others**               | n/a               | dedicated bridge packages                                                                  |

> Mind this when bumping to L9: if an environment uses Mailgun or Postmark, the upgrade **breaks silently** until you add the bridge package. SMTP and SES are fine with what you already have.

---

## 5. Syntax / API differences

### The underlying message object
- Swiftmailer: `Swift_Message` (mutable, `set*()` API).
- Symfony Mailer: `Symfony\Component\Mime\Email` (fluent builder).

```php
// Swiftmailer
$m = new \Swift_Message('Subject');
$m->setFrom(['a@x.com' => 'A'])
  ->setTo(['b@y.com'])
  ->setBody('<p>hi</p>', 'text/html')
  ->addPart('hi', 'text/plain')
  ->attach(\Swift_Attachment::fromPath('/f.pdf'));

// Symfony Mailer
use Symfony\Component\Mime\Email;
$email = (new Email())
    ->subject('Subject')
    ->from(new \Symfony\Component\Mime\Address('a@x.com', 'A'))
    ->to('b@y.com')
    ->text('hi')
    ->html('<p>hi</p>')
    ->attachFromPath('/f.pdf');
```

### Laravel-level API renames (what actually shows up in app code)

| Concern | Swiftmailer (≤L8) | Symfony Mailer (L9+) |
|---|---|---|
| **Mailable raw-message hook** | `->withSwiftMessage(fn (\Swift_Message $m) => …)` | **`->withSymfonyMessage(fn (\Symfony\Component\Mime\Email $m) => …)`** |
| **Get message in `Message`** | `$message->getSwiftMessage()` | direct `Symfony\…\Email` (no wrapper getter) |
| **Custom headers** | `$m->getHeaders()->addTextHeader('X-Foo','v')` | `$m->getHeaders()->addTextHeader('X-Foo','v')` *(same call, but `$m` is now an `Email`)* |
| **`MessageSending` event** | `$event->message` is `Swift_Message` | `$event->message` is `Symfony\…\Email` |
| **`MessageSent` event** | `$event->message` is `Swift_Message` | `$event->message` is `Email`; new `$event->sent` (`SentMessage`) for the wire result |
| **Custom transport** | `Mail::extend('x', fn () => new MySwiftTransport())` returning `Swift_Transport` | `Mail::extend('x', fn () => new MyTransport())` returning **`Symfony\Component\Mailer\Transport\TransportInterface`** |
| **Delivery failures** | `Mail::failures()` | **removed** |
| **Plugin system** | `Swift_Plugins_*` registered on the mailer | **removed** — use Symfony Mailer **event listeners** (`MessageEvent`) |
| **Inline embedding** | `$message->embed(\Swift_Image::fromPath($p))` | Laravel `Attachment` API / `$message->embed($p)` (cid returned) |
| **Attachments** | `attach`, `attachData(\$data,\$name,['mime'=>…])` | same Laravel methods; underneath uses `Email::attach()`/`attachFromPath()` |

### Things that **don't** change (framework abstractions insulate you)
- **Mailables** (`Illuminate\Mail\Mailable`) — `build()`, `->subject()`, `->markdown()`, `->view()`, `->attach()`, `->cc()`, `->bcc()` — all identical.
- **Notifications** via the `mail` channel using `MailMessage` — `->line()`, `->action()`, `->greeting()`, `->attach()` — identical.
- Markdown mail components / themes (`config/mail.php` `markdown.theme`) — identical.
- `Mail::to(...)->send(...)`, `Mail::raw(...)`, `Mail::queue(...)` — identical.

> Rule of thumb: if you only use Mailables, Notifications, and the `Mail` facade, the switch is invisible. Breakage lives entirely in the **raw Swift layer** (the rows above with `Swift_*`).

---

## 6. Migration checklist (general)

1. Bump `laravel/framework` to `^9` → Symfony Mailer comes in transitively; **remove** any explicit `swiftmailer/swiftmailer` requirement.
2. Search the codebase for: `Swift_`, `withSwiftMessage`, `getSwiftMessage`, `Mail::failures(`, `Mail::extend(`, `Swift_Plugins`. Refactor each per the table above.
3. If using **Mailgun/Postmark/Resend**, add the matching Symfony bridge + `symfony/http-client`. SMTP/SES need nothing extra.
4. Verify SMTP `encryption`/port (587 STARTTLS vs 465 TLS) — Symfony auto-negotiates but confirm against your provider.
5. Send a test of each mail type on staging; check headers, attachments, inline images, and DKIM if configured.

---

## 7. This codebase's exposure — near-zero ✅

Audited: **no `Swift_*`, no `withSwiftMessage()`/`getSwiftMessage()`, no custom `Mail::extend()` transports, no `MessageSent` listeners, and zero Mailables.** All mail is sent through **16 Notifications** via the `mail` channel + `MailMessage` — a layer fully insulated from the transport swap. `config/mail.php` is already the modern `mailers` format. The only Swift-coupled dependency, `eduardokum/laravel-mail-auto-embed` (it used a Swift plugin), is **unused and removed in Phase 0**.

**Action required for the switch: essentially none — verification only.** Default transport is `smtp` (SES + Mailgun also configured). SMTP and SES carry over with no extra packages; add `symfony/mailgun-mailer` + `symfony/http-client` **only if** an environment actually sets `MAIL_MAILER=mailgun`. Then send the verify-email, invite, and weekly-report (with its Highcharts payload) notifications on staging to confirm.

---

### Sources
- [The end of Swiftmailer — Symfony Blog](https://symfony.com/blog/the-end-of-swiftmailer)
- [Swiftmailer EOL — Drupal.org issue #3232819](https://www.drupal.org/project/swiftmailer/issues/3232819)
- [Symfony — endoflife.date](https://endoflife.date/symfony)
- [Symfony Release Process & calendar](https://symfony.com/doc/current/contributing/community/releases.html)
- [symfony/mailer — Packagist](https://packagist.org/packages/symfony/mailer)
- [Laravel 9 Upgrade Guide — Symfony Mailer](https://laravel.com/docs/9.x/upgrade)
- [Laravel Mail docs](https://laravel.com/docs/10.x/mail)
