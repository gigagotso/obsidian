# LiveCaller Voice Infrastructure — Architecture & Naming Decisions

A consolidated record of the design discussion for routing browser WebRTC (WSS) and SIP phone traffic into LiveCaller's voice backend, including multi-tenant considerations and DNS naming conventions.

---

## 1. The Core Problem

LiveCaller needs to support two fundamentally different classes of voice client:

1. **Browser clients** using WebRTC, connecting via WSS on port 8089 (or behind nginx on 443)
2. **SIP endpoints** (softphones, hardphones) connecting via UDP/TCP 5060 or TLS 5061

These two paths cannot share one domain when backends differ, and they cannot share the same ingress technology — nginx can terminate WSS but cannot intelligently proxy SIP (UDP is not hostname-aware at the transport layer, and even nginx's stream module only works for TLS-SIP via SNI).

The architecture must therefore split cleanly into two ingress paths while sharing as much downstream infrastructure as possible.

---

## 2. The Two-Path Architecture

### Path 1: Browser → nginx → Voice backend (WSS)

```
Browser (SIP.js / JsSIP)
    ↓ WSS 443
nginx (TLS termination)
    ↓ ws://voice-internal:8088
Voice backend (HTTP module, ws transport)
```

**Key design choices:**

- **Terminate TLS at nginx, talk plain WS internally.** Double-encryption adds CPU cost and complicates cert management. The voice backend's HTTP module binds to an internal interface only; nginx is the only thing that talks to it.
- **WebSocket upgrade headers are mandatory** in nginx config (`Upgrade`, `Connection: upgrade`).
- **Long timeouts** (`proxy_read_timeout 3600s`) — WebSocket connections are long-lived and idle for stretches; default 60s will drop registered clients.

### Path 2: SIP endpoints → Voice backend directly

```
Softphone / Hardphone
    ↓ UDP/TCP 5060 (or TLS 5061)
Voice backend (PJSIP transports)
```

No nginx in this path. SIP clients connect straight to the voice server's public IP on the standard SIP ports.

### Why not put SIP behind nginx?

- nginx is not SIP-aware; it cannot read SIP headers, handle registrations, or do call-aware routing
- UDP SIP cannot be multiplexed by hostname (no SNI in UDP)
- TLS SIP can be proxied via nginx's stream module with `ssl_preread`, but this only solves the multiplexing problem for one transport (TLS), not for UDP/TCP plaintext SIP
- A proper SIP proxy (Kamailio/OpenSIPS) is the correct tool when a proxy layer is needed — but that's a future decision, not a now decision

### RTP — flows direct, never through proxies

A critical detail that often gets missed: **signaling and media are separate.** Browsers do WebRTC media (SRTP/DTLS) over high UDP ports negotiated via SDP. Only signaling flows through nginx; the actual audio goes browser ↔ voice backend directly.

Requirements:
- Voice backend needs a public IP (or 1:1 NAT) reachable from clients on the RTP port range
- `rtp.conf`: set `rtpstart` / `rtpend` (typically 10000-20000)
- Firewall must allow inbound UDP on the RTP range
- PJSIP transport config: `external_media_address` and `external_signaling_address` set to the public IP, `local_net` for the internal subnet
- PJSIP endpoints: `rtp_symmetric=yes`, `force_rport=yes`, `rewrite_contact=yes` for clients behind NAT

---

## 3. Multi-Tenant Considerations (Evaluated and Set Aside)

We considered multi-tenant routing where multiple voice backend servers serve different customers. Three approaches surfaced:

| Approach | Routing axis | Complexity |
|---|---|---|
| Hostname-based | DNS resolves to tenant's backend | Low |
| Credential-based | SIP proxy looks up user → routes | High |
| Port-based | Each tenant on different port | Don't do this |

For multi-backend, multi-tenant, a SIP proxy (Kamailio) becomes essentially required for SIP — there's no clean way to route one SIP hostname to multiple isolated backends without a SIP-aware proxy. WSS can be routed by `Host` header in nginx, but SIP cannot.

**Decision: park multi-tenant for now.** Single backend, single hostname pair, grow into a proxy when scale demands it. The architecture below is single-backend-ready and proxy-ready when the time comes.

---

## 4. Naming Conventions — The Long Discussion

### Constraints

1. **One subdomain pair for all customers** — no per-tenant hostnames. Tenant identity lives in SIP credentials, not DNS.
2. **No technology disclosure** — hostnames must not reveal Asterisk, nginx, or any other component. This is both a security posture (don't help fingerprinters) and a flexibility posture (don't lock the brand to a specific tech stack).
3. **Two transports, one architecture** — WSS for browsers, SIP for phones, but both serve the same voice product.

### Customer-facing hostname structure

Two hostnames are needed: one for the browser/WSS path, one for the SIP path. The question is what to call them.

#### Option set — transport vs role ordering

| Pattern | WSS hostname | SIP hostname |
|---|---|---|
| Transport-first | `ws-pbx1` | `sip-pbx1` |
| Role-first | `pbx-ws1` | `pbx-sip1` |
| Server-first | `pbx1-ws` | `pbx1-sip` |
| Dot-separated | `pbx1.ws` | `pbx1.sip` |
| Subdomain transport | `ws.pbx1` | `sip.pbx1` |

#### Reasoning: transport-first vs role-first

**Transport-first (`ws-pbx1`, `sip-pbx1`)** groups by transport in alphabetical listings — all WS endpoints cluster, all SIP endpoints cluster. The mental model is "the WS endpoint of pbx1." Treats transport as the primary axis.

**Role-first (`pbx-ws1`, `pbx-sip1`)** groups by server role — all PBX endpoints cluster regardless of transport. The mental model is "the WS variant of pbx1." Treats server identity as primary, transport as descriptor.

**The deciding insight:** in this setup, the server is the unit. A "pbx" box serves both WS and SIP simultaneously — they're two faces of the same machine, not two separate systems. Both hostnames resolve to (or proxy to) the same underlying voice server.

That makes role-first more honest. `pbx-ws1` and `pbx-sip1` describe one server with two access paths. `ws-pbx1` and `sip-pbx1` suggest two separate systems, which they aren't.

Additionally, the infrastructure will grow into other **roles** before it grows into other **transports**. Future hostnames will include `db-*`, `app-*`, `cache-*`, `rec-*`, `api-*` long before a third voice transport appears. Role-first scales with how infrastructure actually grows.

**Verdict: role-first.** `pbx-ws*` and `pbx-sip*`.

#### Numbering style

With the role-transport ordering settled, the remaining question is how to number instances:

```
pbx-ws1       ← no padding, no inner separator
pbx-ws01      ← padded, no inner separator
pbx-ws-01     ← padded, inner separator
pbx-ws        ← no number until needed
```

Each has trade-offs:

| Style | Length | Sorts past 9? | Reads cleanly | Industry precedent |
|---|---|---|---|---|
| `pbx-ws1` | shortest | No (breaks at 10) | Yes | Common in small ops |
| `pbx-ws01` | medium | Yes (to 99) | Slight blur (`ws01`) | Cloudflare: `lhr01`, `fra02` |
| `pbx-ws-01` | longest | Yes (to 99) | Cleanest separation | Google SRE examples |
| `pbx-ws` | shortest | Need rename later | Cleanest | Small-scale startups |

#### How other providers actually name things

Two philosophies dominate the industry:

**"Servers are pets" — name them, pad numbers.** Used by smaller ops teams, telcos, traditional enterprise. Pattern: `role-NN` with 2-digit padding. Caps around 99 per role. Examples:
- Bandwidth.com: `gw1.iad.bandwidth.com`, `gw2.iad...`
- Cloudflare edge: `lhr01`, `fra02` (airport-code + padded number)
- Google SRE book: `bigtable-prod-aa-01`
- Facebook historically: `web01.frc1`

**"Servers are cattle" — single public hostname, hide the fleet.** Used by hyperscalers and modern SaaS. One endpoint, anycast or LB-routed internally. Backend names are auto-generated IDs nobody types. Examples:
- Twilio: `sip.twilio.com` — single endpoint, internal routing
- Telnyx: `sip.telnyx.com` — anycast
- Vonage: hidden behind one hostname + internal LB
- Netflix: `i-0abc123...` instance IDs, no human-readable pattern

LiveCaller is firmly in **camp 1** for the foreseeable future — small fleet, hand-managed, hostnames matter. Realistic ceiling for the next 3-5 years: 5-15 voice boxes total.

#### Final numbering decision

For LiveCaller's scale, three reasonable choices:

1. **`pbx-ws01`** — Cloudflare-style middle ground. Padded to 99, no extra dash, 22 characters total with `.livecaller.io`. Real industry precedent.
2. **`pbx-ws-01`** — Google SRE style. Cleanest visual parsing, 25 chars. Best for consistency with future roles (`db-primary-01`, `app-web-01`).
3. **`pbx-ws1`** — Shortest. Fine until box #10 forces a rename or sort-order glitch.

**Recommended: `pbx-ws01` + `pbx-sip01`.**

Rationale:
- Padded from day one — no rename when crossing 9→10
- Matches actual industry precedent (Cloudflare POP naming)
- Shorter than the double-dash variant
- 22 characters total — not too long for ops typing it daily
- Sorts correctly to 99, which is well beyond realistic scale

---

## 5. Final Customer-Facing Names

```
pbx-ws01.livecaller.io       ← browser WSS endpoint (port 443)
pbx-sip01.livecaller.io      ← SIP endpoint (5060 UDP/TCP, 5061 TLS)
```

When the second voice server arrives:
```
pbx-ws02.livecaller.io
pbx-sip02.livecaller.io
```

### Customer setup looks like

**Browser (WebRTC):**
```
WSS URL:  wss://pbx-ws01.livecaller.io/ws
Domain:   livecaller.io
Username: <assigned credentials>
```

**SIP phone:**
```
Server:   pbx-sip01.livecaller.io
Port:     5060 (UDP) or 5061 (TLS, recommended)
Domain:   livecaller.io
Username: <assigned credentials>
```

---

## 6. Internal Hostname Conventions

Internal (non-customer-facing) boxes follow a parallel but distinct convention since they don't need brand polish but do need ops-friendliness.

Pattern: `<role>-<env>-<NN>.internal`

```
pbx-prod-01.internal         ← voice/call processing box
pbx-prod-02.internal         ← second voice box when added
edge-prod-01.internal        ← TLS/WSS terminator (the nginx box)
db-prod-01.internal          ← MySQL
cache-prod-01.internal       ← Redis
app-prod-01.internal         ← Laravel/Vue app server
rec-prod-01.internal         ← recordings storage
api-prod-01.internal         ← REST API server (if separated)
```

Notes:
- Internal names can use `pbx-prod-01` (single role, no transport distinction) because internal hostnames represent the **box**, not the access path. The same box is `pbx-ws01.livecaller.io` and `pbx-sip01.livecaller.io` from outside, but internally it's just `pbx-prod-01`.
- Role names are technology-neutral: `edge` instead of `nginx`, `pbx` instead of `asterisk`, `cache` instead of `redis`. Future migrations (e.g., Asterisk → FreeSWITCH) don't break naming.
- `env` field distinguishes `prod`, `staging`, `dev`.

---

## 7. Technology Disclosure Hardening

Beyond hostnames, several places leak the underlying tech stack to outsiders. These should be sealed:

**nginx (edge):**
```nginx
server_tokens off;                    # Hides nginx version in error pages
more_clear_headers Server;            # Removes "Server: nginx/x.y.z" entirely
# Or rebrand: more_set_headers "Server: LiveCaller";
```

**Asterisk SIP User-Agent:**
```ini
; pjsip.conf — global section
[global]
type=global
user_agent=LiveCaller
```

This rewrites the `User-Agent:` header in outgoing SIP messages. Scanners fingerprinting by User-Agent see "LiveCaller" instead of "Asterisk 20.x.y" — hiding both the product and the version.

**Asterisk HTTP module:**
```ini
; http.conf
[general]
servername=LiveCaller                 ; Replaces "Asterisk/x.y.z" in HTTP responses
```

Combined effect: external scans show generic "LiveCaller" branding everywhere, no version disclosure, no implementation hints.

---

## 8. SIP Identity Conventions

Even on a single dedicated backend today, build SIP usernames to carry tenant context from day one. This avoids any migration pain when consolidating or sharing infrastructure later.

Pattern: `<tenant-slug>.<extension>`

Examples:
```
procredit.100
procredit.200
megatechnica.100             ← no collision with procredit.100
revenue-service.100
```

Alternative if dots cause issues in some clients: `<tenant-slug>-<extension>`:
```
procredit-100
megatechnica-100
```

Lean toward dot-separated — it reads as "extension 100 at procredit" naturally, and modern SIP clients handle it.

For multi-device users (same person on browser + desk phone + mobile softphone), suffix the device type:
```
procredit.100.web            ← browser endpoint
procredit.100.desk           ← hardphone
procredit.100.mobile         ← softphone
```

Or use a single AOR with `max_contacts=5` and let PJSIP fork the INVITE to all registered devices simultaneously.

---

## 9. Dialplan / Context Naming

Inside Asterisk, dialplan contexts should always carry tenant prefix even on a single-tenant box:

```
[procredit-internal]         ; internal calls within tenant
[procredit-outbound]         ; outbound through tenant's SIP trunk
[procredit-inbound]          ; inbound DIDs for this tenant
[procredit-features]         ; feature codes (*XX)
```

Never have a context simply called `internal` or `outbound` — the day a second tenant gets added to the same box (or two boxes get consolidated), context collisions become a migration headache.

---

## 10. Tenant Slug Rules

For the day multi-tenant arrives, lock down the tenant slug format now:

| Rule | Example | Why |
|---|---|---|
| Lowercase only | `bigbank` not `BigBank` | DNS case-insensitive, configs aren't — avoid ambiguity |
| Alphanumeric + hyphens | `big-bank-ge` | DNS-safe, URL-safe, config-safe |
| No underscores | not `big_bank` | Underscores invalid in DNS hostnames |
| 3-20 chars | not `a`, not 50-char monsters | Practical limits |
| No reserved words | not `api`, `admin`, `www`, `pbx`, `sip` | Avoid future collisions |
| Stable, never reused | retired tenant = retired slug | Audit trail clarity |
| Transliterate Georgian | `procredit` not `პროკრედიტი` | DNS doesn't speak Mkhedruli |

Regex: `^[a-z][a-z0-9-]{2,19}$`

---

## 11. White-Label / Customer-Branded Variant

Enterprise customers (banks especially) will eventually want their own URLs. Architecture should support this from day one via CNAME aliases:

```
chat.bigbank.ge    CNAME   pbx-ws01.livecaller.io
voip.bigbank.ge    CNAME   pbx-sip01.livecaller.io
```

For WSS, nginx needs a matching `server_name` block (or wildcard with SNI) and a cert for the customer domain. For SIP, the hostname is cosmetic at the protocol level — clients authenticate by credentials, not hostname, so it works automatically as long as DNS resolves.

Even if this isn't used immediately, build the nginx config to allow additional `server_name` directives without restructuring. Banks will ask within the first year.

---

## 12. Summary — The Final Set

```
Customer-facing hostnames:
  pbx-ws01.livecaller.io          browser WSS endpoint
  pbx-sip01.livecaller.io         SIP endpoint (UDP/TCP/TLS)

Internal hostnames:
  pbx-prod-01.internal            voice processing box
  edge-prod-01.internal           TLS/WSS terminator
  db-prod-01.internal             database
  app-prod-01.internal            web app server
  cache-prod-01.internal          Redis
  rec-prod-01.internal            recordings storage

SIP identity pattern:
  <tenant-slug>.<extension>       e.g. procredit.100
  <tenant>.<extension>.<device>   e.g. procredit.100.web

Dialplan contexts:
  <tenant>-<purpose>              e.g. procredit-internal

Tech disclosure hardening:
  nginx:   server_tokens off, Server header stripped
  voice:   user_agent=LiveCaller in pjsip.conf
  HTTP:    servername=LiveCaller in http.conf

Customer branding (CNAME aliases when needed):
  chat.customer.tld   CNAME   pbx-ws01.livecaller.io
  voip.customer.tld   CNAME   pbx-sip01.livecaller.io
```

---

## 13. Decisions Deferred

These were discussed but explicitly deferred until needed:

- **Kamailio (or other SIP proxy) introduction** — only needed when one SIP hostname must route to multiple isolated backends, when central edge auth/rate-limiting is required, or when tenant-to-backend mobility matters.
- **rtpengine deployment** — only needed when voice backends must be private (not directly reachable for RTP) or when tenant scale exceeds ~10 dedicated boxes.
- **Multi-region / geographic routing** — current scope is single-region (AWS Frankfurt). When expanding, region codes go in hostnames: `pbx-ws-eu01`, `pbx-ws-us01`.
- **HA pairing** — when primary/secondary pairs are added, letter suffixes work cleanly: `pbx-ws01a`, `pbx-ws01b` for active/standby.
- **True multi-tenant on shared backend** — currently single-backend; the SIP username convention (`tenant.extension`) is in place to enable this without rename pain.

---

## 14. Quick Reference — Why These Choices

| Decision | Why |
|---|---|
| Two ingress paths (nginx WSS + direct SIP) | nginx can't proxy SIP intelligently; SIP needs a SIP-aware tool or direct access |
| TLS terminates at nginx, not at voice backend | Single cert location, no double-encryption, easier rotation |
| RTP flows direct to voice backend | Media bypass is fundamental to WebRTC/SIP — proxies handle signaling only |
| `pbx-ws01` over `ws-pbx01` | Server is the unit; transport is the access path. Role-first reflects that. |
| Padded `01` not bare `1` | Sorts correctly past 9; Cloudflare-precedent; future-proof |
| `pbx-ws01` not `pbx-ws-01` | Slightly shorter, still padded, real industry precedent |
| Generic role names (`pbx`, `edge`) not tech names (`asterisk`, `nginx`) | Don't disclose stack; survive tech migrations |
| Tenant prefix in SIP usernames from day one | No migration pain when consolidating to shared infrastructure |
| Single subdomain for all customers | Customer onboarding doesn't require DNS changes; tenant ID in credentials |
| CNAME-able for white-label | Enterprise customers will ask; architecture supports it without rework |
