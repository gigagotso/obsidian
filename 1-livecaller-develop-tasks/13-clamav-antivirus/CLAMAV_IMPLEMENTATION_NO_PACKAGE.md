# ClamAV Integration — Zero-Dependency Implementation Guide (PHP 7.4)

> **Constraint:** No new Composer packages. We talk to the ClamAV daemon
> (`clamd`) directly over its native TCP socket protocol using **PHP core only**
> (`fsockopen`, `fread`, `pack`). Nothing is added to `composer.json`.

> **Companion doc:** `CLAMAV_INTEGRATION_PLAN.md` (research/architecture). This
> file is the step-by-step build with copy-paste-ready PHP 7.4 code.

- **Stack:** PHP 7.4-FPM, Laravel 7, Spatie MediaLibrary 7.19, Redis queue, S3.
- **Container:** `livecaller-php-api-1`, app at `/var/www/api/current`.
- **Integration point:** `App\Actions\Conversation\SendMessage` validation (~line 107).

---

## 0. Why no package is needed

The only thing a package like `xenolope/quahog` does is speak clamd's wire
protocol. That protocol is small and stable:

- Open a TCP connection to `clamd` on port `3310`.
- Send `zINSTREAM\0`, then the file in length-prefixed chunks, then a zero chunk.
- Read one reply line: `stream: OK`, `... FOUND`, or `... ERROR`.

`fsockopen()` + `pack()` are **PHP core** (no extension required — the
`php7.4-sockets` extension is for the lower-level `socket_*` API and is *not*
needed here). So we implement the ~40 lines ourselves and own the code.

---

## 1. Implementation checklist (order of work)

1. **Infra** — add the `clamav` Docker service + config files. *(no code)*
2. **Scanner** — `app/Services/Antivirus/ScanResult.php` + `ClamAvScanner.php`.
3. **Config** — `config/clamav.php` + `.env` vars.
4. **Rule** — `app/Rules/ClamAvClean.php`.
5. **Wire-in** — one edit in `SendMessage.php`.
6. **Remote URLs** — handle `attachmentUrls` (download-then-scan).
7. **Test** — EICAR smoke test + feature test.
8. **Rollout** — shadow → enforce.

All PHP below is written for **PHP 7.4** (no constructor promotion, no
`str_ends_with`, no enums, no union types).

---

## 2. Step 1 — Docker: add the ClamAV service

No package here either — we use the official prebuilt image (`clamd` +
`freshclam` auto-updater bundled).

Edit **`/Users/giga/Developer/livecaller/docker-api/docker-compose.yml`** (the
file that already defines `php-api`, `nginx`, `mysql`, `redis`, `soketi`):

```yaml
services:
  # ... existing services ...

  clamav:
    image: clamav/clamav:1.3      # pin a version; never :latest in prod
    restart: unless-stopped
    networks:
      - default                   # same network as php-api → reachable as "clamav"
    volumes:
      - clamav_data:/var/lib/clamav            # persist virus DB across restarts
      - ./clamav/clamd.conf:/etc/clamav/clamd.conf:ro
    healthcheck:
      test: ["CMD-SHELL", "clamdcheck.sh || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s          # first boot downloads ~250MB signature DB
    deploy:
      resources:
        limits:
          memory: 2g              # clamd keeps the full signature set in RAM

volumes:
  # ... existing volumes ...
  clamav_data:
```

Create **`docker-api/clamav/clamd.conf`**:

```conf
TCPSocket 3310
TCPAddr 0.0.0.0

# MUST be >= your largest allowed upload. Single attachment cap is 15MB,
# MediaLibrary max is 50MB. Set well above.
StreamMaxLength 64M
MaxFileSize 64M
MaxScanSize 128M

# Archive safety (zip bombs)
MaxRecursion 16
MaxFiles 10000

LogTime yes
ExtendedDetectionInfo yes
```

> **Do not publish port 3310 to the host in production** — keep clamd internal to
> the Docker network. clamd has no authentication.

Bring it up and confirm it's healthy:

```bash
cd /Users/giga/Developer/livecaller/docker-api
docker compose up -d clamav
docker compose ps clamav                       # wait for "healthy"
docker exec -it livecaller-php-api-1 bash -lc 'printf "zPING\0" | nc clamav 3310'   # → PONG
```

---

## 3. Step 2 — The scanner (pure PHP 7.4)

### 3.1 `app/Services/Antivirus/ScanResult.php`

A small value object describing a verdict. PHP 7.4 typed properties, explicit
constructor (no promotion).

```php
<?php

namespace App\Services\Antivirus;

class ScanResult
{
    /** @var bool */
    public $clean;
    /** @var bool */
    public $infected;
    /** @var bool */
    public $errored;
    /** @var string|null */
    public $signature;
    /** @var string */
    public $raw;

    public function __construct(bool $clean, bool $infected, bool $errored, ?string $signature, string $raw)
    {
        $this->clean     = $clean;
        $this->infected  = $infected;
        $this->errored   = $errored;
        $this->signature = $signature;
        $this->raw       = $raw;
    }

    /**
     * Parse a raw clamd reply line.
     *  "stream: OK"
     *  "stream: Eicar-Test-Signature FOUND"
     *  "<message> ERROR"
     */
    public static function fromClamdResponse(string $response): self
    {
        $response = trim($response);

        if (self::endsWith($response, 'OK')) {
            return new self(true, false, false, null, $response);
        }

        if (self::endsWith($response, 'FOUND')) {
            // strip "stream:" prefix and " FOUND" suffix → signature name
            $sig = trim(str_replace(['stream:', 'FOUND'], '', $response));
            return new self(false, true, false, $sig, $response);
        }

        // anything else (incl. "... ERROR") is treated as a scan error
        return new self(false, false, true, null, $response);
    }

    private static function endsWith(string $haystack, string $needle): bool
    {
        // PHP 7.4 has no str_ends_with()
        $len = strlen($needle);
        if ($len === 0) {
            return true;
        }
        return substr($haystack, -$len) === $needle;
    }
}
```

### 3.2 `app/Services/Antivirus/ClamAvScanner.php`

Implements the INSTREAM protocol with `fsockopen`. No extension, no package.

```php
<?php

namespace App\Services\Antivirus;

use RuntimeException;

class ClamAvScanner
{
    /** @var string */
    private $host;
    /** @var int */
    private $port;
    /** @var int */
    private $timeout;
    /** @var int bytes per chunk sent to clamd */
    private $chunkSize = 8192;

    public function __construct(string $host, int $port, int $timeout)
    {
        $this->host    = $host;
        $this->port    = $port;
        $this->timeout = $timeout;
    }

    /**
     * Scan an open, readable stream resource.
     *
     * @param resource $stream
     * @throws RuntimeException if clamd is unreachable / connection fails
     */
    public function scanStream($stream): ScanResult
    {
        if (!is_resource($stream)) {
            throw new RuntimeException('ClamAvScanner::scanStream expects an open stream resource.');
        }

        $errno  = 0;
        $errstr = '';
        $socket = @fsockopen('tcp://' . $this->host, $this->port, $errno, $errstr, $this->timeout);

        if ($socket === false) {
            throw new RuntimeException("clamd unreachable at {$this->host}:{$this->port} — {$errstr} ({$errno})");
        }

        stream_set_timeout($socket, $this->timeout);

        try {
            // Begin an in-stream scan.
            fwrite($socket, "zINSTREAM\0");

            // Send the file as <4-byte big-endian length><bytes> chunks.
            while (!feof($stream)) {
                $chunk = fread($stream, $this->chunkSize);
                if ($chunk === false) {
                    break;
                }
                if ($chunk === '') {
                    continue;
                }
                fwrite($socket, pack('N', strlen($chunk)) . $chunk);
            }

            // Zero-length chunk = end of stream.
            fwrite($socket, pack('N', 0));

            // Read the single reply line.
            $response = fgets($socket);

            // Detect a socket-level timeout (clamd never answered).
            $meta = stream_get_meta_data($socket);
            if (!empty($meta['timed_out'])) {
                throw new RuntimeException('clamd read timed out.');
            }

            if ($response === false || trim((string) $response) === '') {
                throw new RuntimeException('clamd returned an empty response.');
            }
        } finally {
            fclose($socket);
        }

        return ScanResult::fromClamdResponse((string) $response);
    }

    /**
     * Convenience: scan a file by path.
     */
    public function scanFile(string $path): ScanResult
    {
        $handle = @fopen($path, 'rb');
        if ($handle === false) {
            throw new RuntimeException("Cannot open file for scanning: {$path}");
        }
        try {
            return $this->scanStream($handle);
        } finally {
            if (is_resource($handle)) {
                fclose($handle);
            }
        }
    }

    /** Liveness check — returns true if clamd answers PONG. */
    public function ping(): bool
    {
        $errno = 0; $errstr = '';
        $socket = @fsockopen('tcp://' . $this->host, $this->port, $errno, $errstr, $this->timeout);
        if ($socket === false) {
            return false;
        }
        stream_set_timeout($socket, $this->timeout);
        fwrite($socket, "zPING\0");
        $reply = trim((string) fgets($socket));
        fclose($socket);
        return $reply === 'PONG';
    }
}
```

> **Memory note:** `scanStream()` streams in 8 KB chunks — it never loads the
> whole file into PHP memory, so a 15 MB upload costs ~8 KB of buffer.

---

## 4. Step 3 — Config + env

Create **`config/clamav.php`**:

```php
<?php

return [
    'enabled' => env('SCAN_UPLOADS_ENABLED', true),
    'host'    => env('CLAMAV_HOST', 'clamav'),
    'port'    => (int) env('CLAMAV_PORT', 3310),
    'timeout' => (int) env('CLAMAV_TIMEOUT', 30),

    // 'block' = fail-closed (reject upload when scan can't complete)
    // 'allow' = fail-open  (accept upload, just log)
    'on_error' => env('CLAMAV_ON_ERROR', 'block'),
];
```

Add to **`.env`** and **`.env.example`**:

```dotenv
SCAN_UPLOADS_ENABLED=true
CLAMAV_HOST=clamav
CLAMAV_PORT=3310
CLAMAV_TIMEOUT=30
CLAMAV_ON_ERROR=block
```

> Run `docker exec -it livecaller-php-api-1 php artisan config:clear` after
> editing env so the new config is picked up.

### 4.1 (Optional) bind the scanner in the container

So you can type-hint `ClamAvScanner` and have Laravel inject it. In
`app/Providers/AppServiceProvider.php`'s `register()`:

```php
$this->app->singleton(\App\Services\Antivirus\ClamAvScanner::class, function () {
    return new \App\Services\Antivirus\ClamAvScanner(
        config('clamav.host'),
        config('clamav.port'),
        config('clamav.timeout')
    );
});
```

This is optional — the rule below can also just `new` it directly.

---

## 5. Step 4 — Validation rule

Create **`app/Rules/ClamAvClean.php`**. This is what makes the scan
**synchronous and pre-storage**: it runs during request validation, so an
infected file never reaches `addMedia()` / S3.

```php
<?php

namespace App\Rules;

use App\Services\Antivirus\ClamAvScanner;
use App\Services\Antivirus\ScanResult;
use Illuminate\Contracts\Validation\Rule;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Log;
use Throwable;

class ClamAvClean implements Rule
{
    /** @var string */
    private $message = 'The :attribute failed a security scan.';

    public function passes($attribute, $value): bool
    {
        // Master kill-switch.
        if (! config('clamav.enabled')) {
            return true;
        }

        // Only scan real uploaded files; let other rules handle the rest.
        if (! $value instanceof UploadedFile || ! $value->isValid()) {
            return true;
        }

        $scanner = new ClamAvScanner(
            config('clamav.host'),
            config('clamav.port'),
            config('clamav.timeout')
        );

        try {
            $result = $scanner->scanFile($value->getRealPath());
        } catch (Throwable $e) {
            Log::error('ClamAV scan failed', [
                'attribute' => $attribute,
                'error'     => $e->getMessage(),
            ]);

            // fail-open vs fail-closed
            if (config('clamav.on_error') === 'allow') {
                return true;
            }
            $this->message = 'This file could not be security-scanned right now. Please try again.';
            return false;
        }

        if ($result->infected) {
            Log::warning('Infected upload blocked', [
                'attribute' => $attribute,
                'signature' => $result->signature,
                'client'    => $value->getClientOriginalName(),
            ]);
            $this->message = 'This file was rejected because it appears to contain malware.';
            return false;
        }

        if ($result->errored && config('clamav.on_error') === 'block') {
            $this->message = 'This file could not be security-scanned right now. Please try again.';
            return false;
        }

        return true; // clean (or errored + fail-open)
    }

    public function message(): string
    {
        return $this->message;
    }
}
```

---

## 6. Step 5 — Wire it into `SendMessage`

In **`app/Actions/Conversation/SendMessage.php`**, the attachments rule is
currently a single string (~**line 107**):

```php
'attachments.*' => 'required|max:' . (1024 * 15) . "|mimes:$extensions|extensions:$extensions",
```

Convert it to array form so you can append the rule object:

```php
'attachments.*' => [
    'required',
    'max:' . (1024 * 15),
    "mimes:$extensions",
    "extensions:$extensions",
    new \App\Rules\ClamAvClean(),
],
```

That's the entire hook. Validation runs **before** the storage loop
(`addMedia(...)->toMediaCollection('attachments')`, ~lines 268–276), so a FOUND
verdict returns `422` and nothing is ever uploaded.

---

## 7. Step 6 — Remote attachments (`attachmentUrls`)

The second loop uses `addMediaFromUrl($attachment['url'])` (~lines 272–276). A
validation rule can't stream a not-yet-downloaded file, so scan it explicitly
before storing. Replace the remote loop body with a download-scan-store flow
(still **no extra package** — uses Laravel's `Http` client which ships with
Laravel 7, or `file_get_contents` to a temp file):

```php
foreach ($data['attachmentUrls'] ?? [] as $attachment) {
    // 1. Download to a temp file.
    $tmp = tempnam(sys_get_temp_dir(), 'dl_');
    file_put_contents($tmp, file_get_contents($attachment['url']));

    try {
        // 2. Scan it.
        if (config('clamav.enabled')) {
            $scanner = new \App\Services\Antivirus\ClamAvScanner(
                config('clamav.host'), config('clamav.port'), config('clamav.timeout')
            );
            $result = $scanner->scanFile($tmp);

            if ($result->infected || ($result->errored && config('clamav.on_error') === 'block')) {
                \Log::warning('Blocked remote attachment', [
                    'url' => $attachment['url'], 'signature' => $result->signature,
                ]);
                continue; // skip this file (or throw to fail the whole message)
            }
        }

        // 3. Only store if clean.
        $message->addMedia($tmp)
            ->preservingOriginal()       // keep the temp file until we delete it
            ->addCustomHeaders($headers)
            ->toMediaCollection('attachments');
    } finally {
        @unlink($tmp);
    }
}
```

> Decide whether a bad remote file should `continue` (silently skip) or `throw`
> (reject the whole message). Match whatever UX you want.

---

## 8. Step 7 — Testing (no package, EICAR)

EICAR is the harmless industry-standard test "virus" string every engine flags.

### 8.1 Smoke test via tinker

```bash
docker exec -it livecaller-php-api-1 php artisan tinker
```

```php
$s = new App\Services\Antivirus\ClamAvScanner('clamav', 3310, 30);
$s->ping();                                   // true
file_put_contents('/tmp/eicar.txt', 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*');
$r = $s->scanFile('/tmp/eicar.txt');
[$r->infected, $r->signature];                // [true, "Eicar-Test-Signature ..."]
$s->scanFile('/etc/hostname')->clean;         // true
```

### 8.2 Feature test (PHPUnit, no package)

`tests/Feature/UploadVirusScanTest.php` — posts EICAR bytes as a fake upload and
asserts a `422` and that no media row was created. (Requires a live `clamav`
container in CI, or mock the bound `ClamAvScanner` singleton to return an
infected `ScanResult`.)

```php
public function test_infected_attachment_is_rejected(): void
{
    // Bind a stub so the test doesn't need a real clamd.
    $this->app->bind(\App\Services\Antivirus\ClamAvScanner::class, function () {
        return new class('x', 0, 0) extends \App\Services\Antivirus\ClamAvScanner {
            public function scanFile(string $path): \App\Services\Antivirus\ScanResult {
                return new \App\Services\Antivirus\ScanResult(false, true, false, 'Eicar-Test', 'stream: Eicar-Test FOUND');
            }
        };
    });

    // ... post a message with an attachment, assert 422 + Media::count() unchanged ...
}
```

> Note: the rule currently `new`s the scanner directly. To make it mockable,
> resolve it from the container instead: `app(ClamAvScanner::class)` (after adding
> the §4.1 binding). Small change, worth it for testability.

---

## 9. Step 8 — Rollout

1. **Infra up:** deploy `clamav`, confirm healthy + `ping()` works from the API
   container. No app behavior change yet.
2. **Shadow mode:** ship the scanner + rule but set `CLAMAV_ON_ERROR=allow`
   (fail-open) and watch logs for verdicts + latency + false positives for a few
   days. Real uploads are unaffected.
3. **Enforce:** flip `CLAMAV_ON_ERROR=block` (fail-closed). Now infected files and
   un-scannable files are rejected. Monitor rejection rate.
4. **Kill switch:** `SCAN_UPLOADS_ENABLED=false` instantly disables scanning if
   anything goes wrong in production.

---

## 10. Operational notes

- **Memory:** clamd needs ~1.5 GB RAM (signatures resident). Limit set to 2 GB.
- **First boot:** ~1–3 min to download the DB; `start_period` covers it.
- **Updates:** `freshclam` runs inside the official image automatically; the
  `clamav_data` volume keeps updates incremental across restarts.
- **`StreamMaxLength`** in `clamd.conf` **must exceed** your max upload (15 MB cap
  today, 50 MB MediaLibrary) or large clean files error out.
- **Config cache:** after editing `config/clamav.php` or env, run
  `php artisan config:clear` (or `config:cache`) in the container.
- **SVG caveat:** `.svg` is in your allowlist and can embed `<script>`; AV may not
  flag it. Keep serving SVGs safely (CSP / `Content-Disposition`) regardless.

---

## 11. Files created / changed (all dependency-free)

| File | New? | Purpose |
|------|------|---------|
| `docker-api/docker-compose.yml` | edit | add `clamav` service + `clamav_data` volume |
| `docker-api/clamav/clamd.conf` | new | daemon config (`StreamMaxLength`, limits) |
| `app/Services/Antivirus/ScanResult.php` | new | verdict value object (PHP 7.4) |
| `app/Services/Antivirus/ClamAvScanner.php` | new | INSTREAM client via `fsockopen` |
| `config/clamav.php` | new | host/port/timeout/fail-policy |
| `.env` / `.env.example` | edit | `CLAMAV_*` + `SCAN_UPLOADS_ENABLED` |
| `app/Providers/AppServiceProvider.php` | edit (optional) | bind scanner singleton |
| `app/Rules/ClamAvClean.php` | new | synchronous pre-storage validation rule |
| `app/Actions/Conversation/SendMessage.php` | edit (~L107 + remote loop) | apply rule + scan remote URLs |
| `tests/Feature/UploadVirusScanTest.php` | new | EICAR / stubbed verdict test |

**`composer.json` is untouched.** Everything uses PHP core (`fsockopen`,
`pack`, `fread`) and the official `clamav/clamav` Docker image.
```
