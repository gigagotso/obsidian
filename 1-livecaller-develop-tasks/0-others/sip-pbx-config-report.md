# SIP / PBX Configuration Report

Differentiating the **Web Call SIP** system from the **PBX phone system**.

> Generated 2026-06-15 from a code sweep of `config/`, `app/Models/`, `app/Services/PBX/`, and `app/Actions/`.

---

## TL;DR — the two systems

| | **Web Call SIP** | **PBX phone system** |
|---|---|---|
| What it is | Browser WebRTC softphones for **website visitors** and **agents** | Asterisk-managed telephony: extensions, softphones, SIP trunks |
| Server identificator | `server_id = 1` → `Endpoint::$webCallServerId` | `server_id = 2` → `Endpoint::$PBXDefaultServerId` (overridable per account via `pbx_server_id` setting) |
| Client connection | `SIP_HOST` → `wss://{SIP_HOST}:8089/ws` (frontend websocket domain) | Managed/controlled server-side via **Asterisk ARI** (`ASTERISK_ARI_HOST`) + **AMI** |
| Control plane | — (browser registers directly over WSS) | ARI (REST) + AMI (events/socket) |
| Defined in | `config/livecaller.php` (`livecaller.sip.*`) | `config/pbx.php` (`pbx.asterisk.ari.*`) + `config/livecaller.php` (`livecaller.ami.*`) |
| DB tables | `ps_endpoints`, `ps_aors`, `ps_auths`, `ps_contacts` filtered by `server_id=1` | same `ps_*` tables filtered by `server_id=2` |

Both systems store endpoints in the **same Asterisk realtime (`ps_*`) tables** — they are separated by the `server_id` discriminator column, **not** by a separate database connection.

---

## 1. Web Call SIP

The browser-based WebRTC softphone used by visitors (web call widget) and agents (dashboard web phone). The frontend connects directly to the SIP server over a secure websocket.

### Configuration — `config/livecaller.php`

| Config key | env var | Default |
|---|---|---|
| `livecaller.sip.host` | `SIP_HOST` | `null` |
| `livecaller.sip.accounts_blacklist` | *(hardcoded)* | `[24, 1067]` |

```php
// config/livecaller.php:4-7
'sip' => [
    'host' => env('SIP_HOST', null),
    'accounts_blacklist' => [24, 1067],
],
```

### Server identificator

`server_id = Endpoint::$webCallServerId = 1` — `app/Models/PBX/Endpoint.php:19`

### Frontend websocket domain

Built as `wss://{SIP_HOST}:8089/ws` and handed to the client SIP stack:

- **Visitor web phone** — `app/Models/Visitor.php:116-120` (`getSipAttribute()`); blacklist check at `:100`
- **Agent web phone** — `app/Models/User.php:140-144` (`getSipAttribute()`), filtered by `where('server_id', Endpoint::$webCallServerId)` at `:134`
- **Account sub-domain / realm** — `app/Models/Account.php:136-139` returns `config('livecaller.sip.host')`

Endpoints are auto-provisioned as a `WebPhone` device (`is_pbx = false` ⇒ `server_id = 1`, transport `wss`) via `app/Actions/CreateSIPDevice.php:83,108`.

### env values

| File | `SIP_HOST` |
|---|---|
| `.env.example` | `"sip.livecaller.io"` |
| `.env` (local) | `"sip.livecaller.test"` |

---

## 2. PBX phone system

The Asterisk-backed telephony system: extensions, desk/softphones, and SIP trunks. The app manages it server-side through Asterisk's REST Interface (ARI) and Manager Interface (AMI) — there is no separate frontend websocket domain config for it.

### a) Asterisk ARI (control / REST) — `config/pbx.php`

| Config key | env var | Default |
|---|---|---|
| `pbx.asterisk.ari.host` | `ASTERISK_ARI_HOST` | *(none)* |
| `pbx.asterisk.ari.port` | `ASTERISK_ARI_PORT` | `8088` |
| `pbx.asterisk.ari.user` | `ASTERISK_ARI_USER` | `admin` |
| `pbx.asterisk.ari.password` | `ASTERISK_ARI_PASSWORD` | *(none)* |
| `pbx.asterisk.ari.endpoint_state_cache_ttl` | *(hardcoded)* | `600` |

- **Consumed:** `app/Services/PBX/AsteriskAriService.php:21-24`
- **Endpoint-state cache TTL:** `app/Models/PBX/Endpoint.php:119`

env values:

| File | `ASTERISK_ARI_HOST` | port | user | password |
|---|---|---|---|---|
| `.env.example` | `http://sip.livecaller.work` | `8088` | `admin` | `password` |
| `.env` (local) | `https://pbx.test.livecaller.io` | `8089` | `admin` | *(set)* |

### b) Asterisk AMI (events / socket) — `config/livecaller.php`

| Config key | env var | Default |
|---|---|---|
| `livecaller.ami.host` | `AMI_HOST` | `127.0.0.1` |
| `livecaller.ami.port` | `AMI_PORT` | `5038` |
| `livecaller.ami.username` | `AMI_USERNAME` | `null` |
| `livecaller.ami.secret` | `AMI_SECRET` | `null` |
| `livecaller.ami.scheme` | *(hardcoded)* | `tcp://` |
| `livecaller.ami.connect_timeout` / `read_timeout` | *(hardcoded)* | `10000` |

- **Consumed:** `app/AMI/AMIConnection.php:18,34,52,63` — no-ops gracefully when host is unset.

### c) Server identificator (PBX)

This is **not env-managed**. It is resolved at runtime:

```php
// app/Actions/CreateSIPDevice.php:83
$serverId = $data['is_pbx']
    ? (!empty($account->settings['pbx_server_id'])
        ? $account->settings['pbx_server_id']
        : Endpoint::$PBXDefaultServerId)   // = 2
    : Endpoint::$webCallServerId;          // = 1
```

- Constant default: `Endpoint::$PBXDefaultServerId = 2` — `app/Models/PBX/Endpoint.php:20`
- Per-account override: `$account->settings['pbx_server_id']`
- Used to scope queries/route-binding: `app/Models/PBX/Endpoint.php:63-70`, `app/Services/PBX/EndpointService.php:126`, `app/Services/PBX/TrunkService.php:132`, `app/Http/Controllers/DashboardController.php:28`

### d) PBX webhook signing — `config/webhook-client.php`

| Config key | env var | Default |
|---|---|---|
| pbx config `signing_secret` | `PBX_SIGNATURE_SECRET` | *(none)* |

- HMAC validation of inbound PBX webhook events — `config/webhook-client.php:59`.

---

## 3. Shared mechanics (important nuance)

- **Same `ps_*` tables, same DB connection.** Web Call (`server_id=1`) and PBX (`server_id=2`) endpoints live together in `ps_endpoints` / `ps_aors` / `ps_auths` / `ps_contacts`; the `server_id` column is the only separator. There is no `ps_`-specific DB connection — they use the default `mysql` connection (`config/database.php`).
- **Realm.** Both web-call and (non-trunk) PBX auths default their `realm` to `$account->sub_domain`, which itself returns `config('livecaller.sip.host')` — see `app/Actions/CreateSIPDevice.php:99` and `app/Models/Account.php:138`. SIP **trunks** get `realm = null`.
- **In current code the frontend websocket host (`SIP_HOST`) is shared** by visitor and agent web phones. The PBX phone system's distinct identity is expressed through the **ARI host + `server_id=2`**, not through a second `SIP_HOST`-style frontend domain. If a physically separate PBX SIP server is desired, it is selected per account via the `pbx_server_id` setting (a different Asterisk `server_id`), while ARI/AMI hosts point the app at that server's control interfaces.

---

## 4. Quick env reference

```dotenv
# --- Web Call SIP (frontend websocket domain) ---
SIP_HOST=sip.livecaller.io            # wss://{SIP_HOST}:8089/ws  (server_id=1)

# --- PBX phone system: Asterisk ARI (control/REST) ---
ASTERISK_ARI_HOST=                    # e.g. https://pbx.test.livecaller.io
ASTERISK_ARI_PORT=8088
ASTERISK_ARI_USER=admin
ASTERISK_ARI_PASSWORD=

# --- PBX phone system: Asterisk AMI (events/socket) ---
AMI_HOST=127.0.0.1
AMI_PORT=5038
AMI_USERNAME=
AMI_SECRET=

# --- PBX webhook signature ---
PBX_SIGNATURE_SECRET=

# --- PBX server identificator: NOT env-managed ---
# Endpoint::$webCallServerId = 1  (Web Call)
# Endpoint::$PBXDefaultServerId = 2  (PBX default; override per account via settings['pbx_server_id'])
```
