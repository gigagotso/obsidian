# ClamAV Antivirus Integration Plan — LiveCaller API

> **Goal:** Scan every user-uploaded file for malware *before* it is stored or
> served, using ClamAV running as a containerized scanning microservice that the
> PHP API talks to over the clamd TCP protocol.

- **Stack:** PHP 7.4-FPM, Laravel 7, Spatie MediaLibrary 7.19, Redis queue, S3 storage.
- **Primary upload path:** `App\Actions\Conversation\SendMessage` → `addMedia()` → `toMediaCollection('attachments')`.
- **Author:** _research/plan — not yet implemented_
- **Status:** DRAFT

---

## 1. Executive Summary

We add **ClamAV** as a dedicated Docker service (the "antivirus microservice").
The PHP API streams each uploaded file to the ClamAV daemon (`clamd`) over TCP
port `3310` using the `INSTREAM` command and only persists the file if the verdict
is `OK`. Infected or errored files are rejected at validation time, so a malicious
file is **never written to S3 and never delivered into a conversation**.

Two integration depths are described:

| Mode                                           | Where it runs                  | Blocks upload?              | Recommended                            |
| ---------------------------------------------- | ------------------------------ | --------------------------- | -------------------------------------- |
| **A. Synchronous validation rule** (primary)   | In the request, before storage | Yes — infected file → `422` | ✅ Yes for chat attachments             |
| **B. Async queued re-scan** (defense-in-depth) | `ScanUpload` job after store   | No                          | Optional, for `attachmentUrls` + audit |

We recommend **Mode A** as the main control because chat attachments are
delivered to the other participant almost immediately — an async-only scan would
let an infected file be served before the verdict arrives.

---

## 2. Why ClamAV + clamd (not a CLI scan per request)

- `clamscan` (CLI) reloads the full ~250 MB signature database on every invocation
  → seconds of latency per file. **Unusable** in the request path.
- `clamd` (daemon) keeps signatures resident in memory and exposes a socket API.
  A 15 MB file scans in **tens of milliseconds**. This is what we use.
- The official `clamav/clamav` image bundles `clamd` **and** `freshclam` (the
  signature auto-updater), so the container is self-maintaining.

---

## 3. Architecture

### 3.1 Component diagram

```
            ┌─────────────────────────────────────────────────────────┐
            │                  docker "default" network                │
            │                                                           │
  Visitor   │   ┌──────────┐   POST /v1/.../messages   ┌────────────┐  │
  / Agent ──┼──▶│  nginx   │ ─────────────────────────▶│  php-api   │  │
   upload   │   └──────────┘                            │ (FPM 7.4)  │  │
            │                                           └─────┬──────┘  │
            │                                                 │         │
            │                  INSTREAM (TCP 3310)            │         │
            │                                                 ▼         │
            │                                           ┌────────────┐  │
            │                                           │   clamav   │  │
            │                                           │ clamd +    │  │
            │                                           │ freshclam  │  │
            │                                           └─────┬──────┘  │
            │                                                 │         │
            └─────────────────────────────────────────────────┼─────────┘
                                                              │
                          verdict OK ──▶ Spatie MediaLibrary ──▶ S3
                          verdict FOUND ──▶ reject (422) + audit log
```

### 3.2 Request flow (Mode A — synchronous)

1. Client `POST`s files to `POST /v1/conversations/{conversation}/messages`
   (widget) or `POST /conversations/{conversation}/messages` (API).
2. `SendMessage` runs Laravel validation. The custom rule **`ClamAvClean`** opens
   each `UploadedFile`'s read stream and `INSTREAM`s it to `clamav:3310`.
3. clamd returns `stream: OK` → validation passes; `... FOUND` → validation fails
   with a user-facing error; the file is **never** passed to `addMedia()`.
4. Only clean files reach `->toMediaCollection('attachments')` and get uploaded
   to S3 by MediaLibrary.

---

## 4. Docker Microservice Setup

### 4.1 Add the `clamav` service

Edit **`/Users/giga/Developer/livecaller/docker-api/docker-compose.yml`** (same
file that defines `php-api`, `nginx`, `mysql`, `redis`, `soketi`). All services
share the `default` network, so the API reaches ClamAV by the service hostname
`clamav`.

```yaml
services:
  # ... existing php-api, nginx, mysql, redis, soketi ...

  clamav:
    image: clamav/clamav:1.3   # pin a version; avoid :latest in prod
    restart: unless-stopped
    networks:
      - default
    ports:
      # Expose only if you need host access for debugging. Internal
      # service-to-service traffic does NOT require a published port.
      - "127.0.0.1:3310:3310"
    volumes:
      # Persist the virus DB so freshclam doesn't re-download ~250MB on every
      # container recreate, and signatures survive restarts.
      - clamav_data:/var/lib/clamav
      # Optional: override default daemon config
      - ./clamav/clamd.conf:/etc/clamav/clamd.conf:ro
      - ./clamav/freshclam.conf:/etc/clamav/freshclam.conf:ro
    healthcheck:
      # clamd answers "PONG" on the socket only once signatures are loaded.
      test: ["CMD-SHELL", "clamdcheck.sh || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s   # first boot downloads the signature DB
    deploy:
      resources:
        limits:
          memory: 2g        # clamd holds the full signature set in RAM (~1.5GB)

volumes:
  # ... existing volumes ...
  clamav_data:
```

> **`php-api` dependency (optional):** add
> ```yaml
> php-api:
>   depends_on:
>     clamav:
>       condition: service_healthy
> ```
> so FPM doesn't start accepting uploads before the AV DB is ready. Weigh this
> against startup coupling — with a graceful fail policy (§8) it's not required.

### 4.2 `clamd.conf` (relevant tunables)

Create **`docker-api/clamav/clamd.conf`**:

```conf
# Listen for the API on TCP
TCPSocket 3310
TCPAddr 0.0.0.0

# Must be >= the largest file you allow. Attachments are capped at 15MB,
# MediaLibrary max is 50MB. Set comfortably above the cap.
StreamMaxLength 64M
MaxFileSize 64M
MaxScanSize 128M

# Reasonable safety limits for archives (zip bombs etc.)
MaxRecursion 16
MaxFiles 10000

# Logging
LogTime yes
LogClean no
ExtendedDetectionInfo yes

# Keep the daemon responsive
ConcurrentDatabaseReload no
```

> **Critical:** `StreamMaxLength` must be **≥ your largest allowed upload**. The
> attachment validation rule caps single files at **15 MB**
> (`'attachments.*' => 'required|max:' . (1024 * 15)`), and MediaLibrary's
> `max_file_size` is **50 MB**. If `StreamMaxLength` is smaller, clamd aborts the
> stream with `INSTREAM size limit exceeded` and the scan errors out.

### 4.3 `freshclam.conf` (signature updates)

Create **`docker-api/clamav/freshclam.conf`**:

```conf
DatabaseMirror database.clamav.net
Checks 24                # update checks per day
NotifyClamd /etc/clamav/clamd.conf   # reload clamd after DB update
```

The official image runs `freshclam` in the background automatically; this file
only customizes cadence/mirrors. The persisted `clamav_data` volume means updates
are incremental after the first download.

### 4.4 Resource & operational notes

- **Memory:** clamd needs ~1–1.5 GB resident for the signature set. Budget 2 GB.
- **First boot:** downloads the full DB (~250 MB) — can take 1–3 minutes. The
  `start_period` + healthcheck cover this; don't route traffic until healthy.
- **CPU:** scanning is CPU-bound but fast for typical chat files (images/PDFs).

---

## 5. The ClamAV "API" — INSTREAM protocol

clamd's TCP protocol is the antivirus API. The relevant commands:

| Command | Purpose |
|---------|---------|
| `zPING\0` | Liveness check → replies `PONG` |
| `zVERSION\0` | Returns engine + DB version |
| `zINSTREAM\0` | Stream a file's bytes for scanning (what we use) |

### 5.1 INSTREAM wire format

1. Send the command: the bytes `zINSTREAM\0` (the `z` prefix = NUL-terminated
   command; `n` prefix = newline-terminated — pick one and be consistent).
2. Send the file as a series of chunks. Each chunk is:
   `<4-byte length, big-endian (network order)><that many bytes>`.
3. Terminate with a **zero-length chunk**: four `0x00` bytes.
4. Read the reply line. Possible replies:
   - `stream: OK` → clean
   - `stream: <SignatureName> FOUND` → infected
   - `<message> ERROR` → scan error (e.g. size limit, DB not loaded)

> Each chunk must be smaller than `StreamMaxLength`; the **sum** of chunks must
> also be ≤ `StreamMaxLength`. Use an 8 KB read buffer.

---

## 6. PHP Client

Two options. **Recommend Option A** (battle-tested library) and fall back to
Option B (zero-dependency) if you'd rather not add a Composer package.

> All `composer` commands run **inside the container**
> (`livecaller-php-api-1`, app at `/var/www/api/current`), never on the host.

### 6.1 Option A — `xenolope/quahog` (recommended)

Mature ClamAV client, supports PHP 7.x.

```bash
docker exec -it livecaller-php-api-1 \
  composer require xenolope/quahog:^3.0
```

Usage:

```php
use Socket\Raw\Factory;
use Xenolope\Quahog\Client;

$socket = (new Factory())->createClient("tcp://{$host}:{$port}", $timeout);
$quahog = new Client($socket, $timeout, PHP_NORMAL_READ);

$result = $quahog->scanResourceStream($fopenHandle); // streams the file
// $result->isOk(), $result->isFound(), $result->getReason() (signature name)
```

### 6.2 Option B — zero-dependency raw socket client

If you prefer no new dependency, this implements INSTREAM directly. Create
**`app/Services/Antivirus/ClamAvScanner.php`**:

```php
<?php

namespace App\Services\Antivirus;

use RuntimeException;

class ClamAvScanner
{
    public function __construct(
        private string $host,
        private int $port,
        private int $timeout
    ) {}

    /**
     * @param resource $stream  An open, readable file handle.
     * @return ScanResult
     */
    public function scanStream($stream): ScanResult
    {
        $socket = @fsockopen("tcp://{$this->host}", $this->port, $errno, $errstr, $this->timeout);
        if ($socket === false) {
            throw new RuntimeException("clamd unreachable: {$errstr} ({$errno})");
        }
        stream_set_timeout($socket, $this->timeout);

        try {
            fwrite($socket, "zINSTREAM\0");

            while (!feof($stream)) {
                $chunk = fread($stream, 8192);
                if ($chunk === '' || $chunk === false) {
                    break;
                }
                // 4-byte big-endian length prefix, then the bytes.
                fwrite($socket, pack('N', strlen($chunk)) . $chunk);
            }

            // Zero-length chunk terminates the stream.
            fwrite($socket, pack('N', 0));

            $response = trim((string) fgets($socket));
        } finally {
            fclose($socket);
        }

        return ScanResult::fromClamdResponse($response);
    }
}
```

**`app/Services/Antivirus/ScanResult.php`**:

```php
<?php

namespace App\Services\Antivirus;

class ScanResult
{
    public function __construct(
        public bool $clean,
        public bool $infected,
        public bool $errored,
        public ?string $signature,
        public string $raw
    ) {}

    public static function fromClamdResponse(string $response): self
    {
        // "stream: OK" | "stream: Eicar-Test-Signature FOUND" | "... ERROR"
        if (str_ends_with($response, 'OK')) {
            return new self(true, false, false, null, $response);
        }
        if (str_ends_with($response, 'FOUND')) {
            // "stream: <Signature> FOUND"
            $sig = trim(str_replace(['stream:', 'FOUND'], '', $response));
            return new self(false, true, false, $sig, $response);
        }
        return new self(false, false, true, null, $response);
    }
}
```

> `str_ends_with` / constructor property promotion shown above are PHP 8 niceties.
> **On PHP 7.4** replace `str_ends_with($s,$x)` with
> `substr($s, -strlen($x)) === $x` and expand the promoted constructor to explicit
> property assignments.

---

## 7. Laravel Integration (Mode A — synchronous validation)

### 7.1 Config file `config/clamav.php`

```php
<?php

return [
    'enabled' => env('SCAN_UPLOADS_ENABLED', true),
    'host'    => env('CLAMAV_HOST', 'clamav'),
    'port'    => (int) env('CLAMAV_PORT', 3310),
    'timeout' => (int) env('CLAMAV_TIMEOUT', 30),

    // 'block' = fail-closed (reject upload if clamd is down)
    // 'allow' = fail-open  (permit upload, log a warning)
    'on_error' => env('CLAMAV_ON_ERROR', 'block'),
];
```

### 7.2 Env vars (add to `.env` / `.env.example`)

```dotenv
SCAN_UPLOADS_ENABLED=true
CLAMAV_HOST=clamav
CLAMAV_PORT=3310
CLAMAV_TIMEOUT=30
CLAMAV_ON_ERROR=block
```

### 7.3 Validation rule `app/Rules/ClamAvClean.php`

```php
<?php

namespace App\Rules;

use App\Services\Antivirus\ClamAvScanner;
use Illuminate\Contracts\Validation\Rule;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Log;
use Throwable;

class ClamAvClean implements Rule
{
    private string $message = 'The :attribute failed a security scan.';

    public function passes($attribute, $value): bool
    {
        if (! config('clamav.enabled')) {
            return true;
        }
        if (! $value instanceof UploadedFile) {
            return true; // let other rules handle non-files
        }

        $scanner = new ClamAvScanner(
            config('clamav.host'),
            config('clamav.port'),
            config('clamav.timeout')
        );

        try {
            $handle = fopen($value->getRealPath(), 'rb');
            $result = $scanner->scanStream($handle);
        } catch (Throwable $e) {
            Log::error('ClamAV scan error', ['error' => $e->getMessage()]);
            // fail-closed vs fail-open, per config
            if (config('clamav.on_error') === 'allow') {
                return true;
            }
            $this->message = 'File could not be security-scanned. Please retry.';
            return false;
        } finally {
            if (isset($handle) && is_resource($handle)) {
                fclose($handle);
            }
        }

        if ($result->infected) {
            Log::warning('Infected upload blocked', [
                'attribute' => $attribute,
                'signature' => $result->signature,
            ]);
            $this->message = 'This file was rejected because it appears to contain malware.';
            return false;
        }

        if ($result->errored && config('clamav.on_error') === 'block') {
            $this->message = 'File could not be security-scanned. Please retry.';
            return false;
        }

        return true;
    }

    public function message(): string
    {
        return $this->message;
    }
}
```

### 7.4 Wire it into `SendMessage`

In **`app/Actions/Conversation/SendMessage.php`** the attachment rule is currently
(around **line 107**):

```php
'attachments.*' => 'required|max:' . (1024 * 15) . "|mimes:$extensions|extensions:$extensions",
```

Append the rule object. Because the line mixes string rules with an object, switch
that entry to array form:

```php
'attachments.*' => [
    'required',
    'max:' . (1024 * 15),
    "mimes:$extensions",
    "extensions:$extensions",
    new \App\Rules\ClamAvClean(),
],
```

Validation runs **before** the storage loop (lines ~268–276), so an infected file
never reaches `addMedia()` / S3.

### 7.5 Remote attachments (`attachmentUrls`)

The second loop uses `addMediaFromUrl($attachment['url'])` (lines ~272–276). A
validation rule can't stream a not-yet-downloaded remote file cleanly, so handle
these one of two ways:

- **Download-then-scan-then-store:** fetch the URL to a temp file, run
  `ClamAvScanner::scanStream()` on it, and only `addMediaFromUrl` (or
  `addMedia($tmp)`) if clean. Reject the whole message otherwise.
- **Async re-scan (Mode B):** allow store, then a queued `ScanUpload` job
  re-scans the stored object and quarantines on a FOUND verdict (§9).

---

## 8. Failure Policy: fail-closed vs fail-open

Decide what happens when **clamd is unreachable or errors**:

| Policy | Behavior | Use when |
|--------|----------|----------|
| **fail-closed** (`CLAMAV_ON_ERROR=block`) | Reject uploads if scan can't complete | Default for production — security over availability |
| **fail-open** (`CLAMAV_ON_ERROR=allow`) | Accept uploads, log a warning + alert | If upload availability is business-critical and you accept the gap |

Recommendation: **fail-closed in production**, with monitoring/alerting on clamd
health so an outage is noticed and fixed fast rather than silently degrading
security. Use fail-open only as a deliberate, temporary escape hatch.

---

## 9. Defense-in-Depth: async re-scan + quarantine (Mode B, optional)

For audit trails and for the `attachmentUrls` path, add a queued job that runs on
the existing **Redis** queue.

### 9.1 Audit table (migration)

```php
Schema::create('upload_scans', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('media_id')->nullable()->index();
    $table->string('verdict');            // clean | infected | error
    $table->string('signature')->nullable();
    $table->string('engine_version')->nullable();
    $table->timestamps();
});
```

### 9.2 Job `app/Jobs/ScanUpload.php`

```php
class ScanUpload implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public int $mediaId) {}

    public function handle(ClamAvScanner $scanner): void
    {
        $media  = Media::findOrFail($this->mediaId);
        $stream = Storage::disk($media->disk)->readStream($media->getPath());
        $result = $scanner->scanStream($stream);

        UploadScan::create([
            'media_id'  => $media->id,
            'verdict'   => $result->infected ? 'infected' : ($result->errored ? 'error' : 'clean'),
            'signature' => $result->signature,
        ]);

        if ($result->infected) {
            // Quarantine: move to an isolated bucket or delete, and redact the
            // message attachment so it's never served.
            Storage::disk($media->disk)->delete($media->getPath());
            $media->delete();
            // TODO: notify the conversation / flag for review.
        }
    }
}
```

Dispatch it after a successful store if you want the audit/defense layer even with
Mode A enabled.

---

## 10. Testing

### 10.1 EICAR test file (safe, industry-standard)

The EICAR string is a harmless file every AV engine flags as a test virus:

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

### 10.2 Smoke test from the API container

```bash
# liveness
docker exec -it livecaller-php-api-1 bash -lc \
  'printf "zPING\0" | nc clamav 3310'        # expect: PONG

# scan EICAR via tinker
docker exec -it livecaller-php-api-1 php artisan tinker
>>> $s = new App\Services\Antivirus\ClamAvScanner('clamav', 3310, 30);
>>> file_put_contents('/tmp/eicar.txt', 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*');
>>> $s->scanStream(fopen('/tmp/eicar.txt','rb'));   // expect infected=true, signature ~ "Eicar-..."
```

### 10.3 End-to-end

- Upload a clean PNG via the widget → message sends, file in S3.
- Upload an EICAR `.txt` (temporarily allow `.txt` for the test account) →
  `422` with the malware message; nothing written to S3.
- Stop the `clamav` container → confirm fail-closed returns the
  "could not be scanned" error (and fail-open lets it through when configured).

### 10.4 Automated test

A feature test that posts EICAR bytes as `UploadedFile::fake()` content and asserts
a `422` + that `Media::count()` did not increase.

---

## 11. Rollout Plan

1. **Phase 0 — Infra:** add `clamav` service + config files; deploy; verify
   healthcheck + `PING`/`VERSION` from `php-api`. No app code yet.
2. **Phase 1 — Client:** add `ClamAvScanner`/`ScanResult` (or quahog),
   `config/clamav.php`, env vars. Cover with unit tests (EICAR + clean).
3. **Phase 2 — Shadow mode:** add the rule with `SCAN_UPLOADS_ENABLED=true` but
   `CLAMAV_ON_ERROR=allow` (fail-open) and only **log** verdicts for a few days —
   measure latency and false positives without blocking users.
4. **Phase 3 — Enforce:** flip to fail-closed in production; wire the rule into
   `SendMessage`. Monitor rejection rates.
5. **Phase 4 — Defense-in-depth (optional):** add `upload_scans` table +
   `ScanUpload` job for `attachmentUrls` and audit history.

---

## 12. Monitoring & Maintenance

- **Health:** scrape clamd `PING`/`VERSION`; alert if unhealthy (drives the
  fail-closed risk window).
- **Signature freshness:** alert if `freshclam` hasn't updated in > 24 h
  (`VERSION` includes DB version/date).
- **Metrics:** count of `clean` / `infected` / `error` verdicts and scan latency.
- **Memory:** watch the container's RSS; clamd grows when the DB reloads.
- **Version pinning:** bump the `clamav/clamav` image tag deliberately; test in
  staging — engine upgrades occasionally change protocol/limits behavior.

---

## 13. Security Considerations

- **Pre-storage rejection** (Mode A) is the strongest control: malware never lands
  in S3 and is never delivered into a conversation.
- **Don't expose port 3310 publicly** — keep it on the internal `default` network
  (the compose snippet binds to `127.0.0.1` only if published at all). clamd has no
  auth; anyone who can reach the socket can scan/learn engine details.
- **Zip-bomb / recursion limits** are set in `clamd.conf` (`MaxRecursion`,
  `MaxFiles`, `MaxScanSize`) to bound resource use on archive uploads.
- **`StreamMaxLength` must exceed the app's max upload** (§4.2) or large clean
  files will spuriously error.
- ClamAV is **signature-based** — it is not a complete defense against novel/
  targeted malware. It catches commodity threats. Keep the extension allowlist
  (`jpeg,png,jpg,gif,svg,pdf`) and size caps as complementary controls.
- **SVG note:** `.svg` is allowed and can carry embedded `<script>`. ClamAV may
  not flag malicious SVG; continue to sanitize/serve SVGs safely
  (CSP, `Content-Disposition`, no inline execution) independent of AV.

---

## 14. Open Questions / Decisions Needed

1. **Sync vs async** as the primary control — recommend **sync (Mode A)** for
   chat attachments. Confirm acceptable added latency (tens of ms typical).
2. **Fail policy** — confirm **fail-closed** in production (§8).
3. **`attachmentUrls`** — download-then-scan inline, or async re-scan? (§7.5)
4. **Quarantine destination** — delete vs. move to an isolated bucket for review.
5. **Client choice** — `xenolope/quahog` package vs. the in-repo raw socket client.
6. **Scope** — attachments only, or also other upload paths (avatars, imports,
   exports) elsewhere in the app?

---

## 15. Files Touched (summary)

| File | Change |
|------|--------|
| `docker-api/docker-compose.yml` | Add `clamav` service + `clamav_data` volume |
| `docker-api/clamav/clamd.conf` | New — daemon config (`StreamMaxLength`, limits) |
| `docker-api/clamav/freshclam.conf` | New — signature update cadence |
| `config/clamav.php` | New — host/port/timeout/fail-policy |
| `.env` / `.env.example` | New `CLAMAV_*` + `SCAN_UPLOADS_ENABLED` vars |
| `app/Services/Antivirus/ClamAvScanner.php` | New — INSTREAM client |
| `app/Services/Antivirus/ScanResult.php` | New — verdict value object |
| `app/Rules/ClamAvClean.php` | New — validation rule |
| `app/Actions/Conversation/SendMessage.php` | Add rule to `attachments.*` (~line 107) |
| `app/Jobs/ScanUpload.php` | New (optional, Mode B) |
| `database/migrations/*_create_upload_scans_table.php` | New (optional, Mode B) |

---

## Appendix A — Optional REST Gateway Microservice

If you'd rather not speak the clamd binary protocol from PHP (or want a
language-agnostic scan API reusable by other services), put a thin HTTP wrapper in
front of clamd:

```
php-api ──HTTP POST /scan (multipart)──▶ av-gateway ──INSTREAM──▶ clamd
                                         (Node/Go)     (TCP 3310)
         ◀── 200 {"clean":true} / {"clean":false,"signature":"..."} ──
```

- **Pros:** PHP only does an HTTP call; central place for retries, metrics,
  rate-limiting; reusable by non-PHP services.
- **Cons:** extra service to build/operate; multipart re-buffers the file (more
  memory/latency) vs. streaming straight to clamd.

Recommendation: **start without the gateway** — talk to clamd directly (§6). Add
the gateway only if a second, non-PHP service later needs scanning too.

The official image already exposes everything needed; the gateway is purely an
ergonomic/abstraction layer, not a functional requirement.
