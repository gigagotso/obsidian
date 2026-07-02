# 02 — Getting Audio Out of an Asterisk Channel

The AI node needs the call's audio to reach an AI pipeline and the AI's speech back to the caller. Asterisk offers several mechanisms. This page compares them so [[05-implementation-plan]] can pick. See also [[00-index]].

Telephony baseline: PJSIP channels carry **G.711 μ-law/A-law @ 8 kHz** (or slin16 after transcode). Everything below is about extracting that stream.

---

## Option A — AudioSocket

Native Asterisk 18+ (`res_audiosocket` / `chan_audiosocket`). Asterisk opens a **raw TCP** connection to your server and streams audio frames; you stream audio back over the same socket. This is what [[03-reference-avr|AVR]] uses and one of [[04-reference-ava|AVA]]'s two modes.

**Dialplan forms:**
```asterisk
; application form (runs on current channel; Asterisk auto-transcodes to slin16)
same => n,Set(UUID=${SHELL(uuidgen | tr -d '\n')})
same => n,AudioSocket(${UUID},MEDIA_GW_HOST:5001)

; dial form (scales better, pre-negotiated codecs)
same => n,Dial(AudioSocket/MEDIA_GW_HOST:5001/${UUID})
```

**Wire protocol** (CyCoreSystems spec): TCP, each packet = **3-byte header + payload**:
- byte 0 = type, bytes 1–2 = payload length (uint16, **big-endian**).
- Types: `0x01` UUID (first frame, 16-byte binary UUID) · `0x10` audio (signed-linear 16-bit, **8 kHz**, mono, little-endian PCM = slin16) · `0x03` DTMF (1 ASCII byte) · `0x00` terminate · `0xff` error.
- Frames are ~20 ms = 320 bytes of PCM at 8 kHz/16-bit.

| ✅ Pros | ⚠️ Cons |
|---|---|
| Dead simple TCP, easy to implement in any language | App form does extra transcode (CPU) |
| UUID ties stream ↔ call session cleanly | Audio only (control via separate ARI/AMI) |
| Works with a remote gateway (network-friendly) | You build resampling 8k↔16k/24k yourself |
| Built-in DTMF frames | No native barge-in; you run VAD |

From ARI you can also originate an AudioSocket channel and add it to a bridge — combining ARI control with AudioSocket media.

---

## Option B — ARI ExternalMedia

ARI `POST /channels/externalMedia` creates a special channel that ships media to an external host (RTP, or AudioSocket encapsulation), then you add it to a bridge with the caller. This is [[04-reference-ava|AVA]]'s primary mode and keeps **ARI fully in control** of the call (bridges, transfers, playback) — which fits `node-asterisk-ari`'s existing model.

```jsonc
POST /channels/externalMedia
{ "app": "incoming", "external_host": "MEDIA_GW:18080",
  "format": "ulaw", "direction": "both", "encapsulation": "rtp" }   // or "audiosocket"
```

Flow: on `StasisStart` → create mixing bridge → create externalMedia channel → add caller + externalMedia channel to the bridge → answer. Media (μ-law/slin RTP) flows to your gateway and back.

| ✅ Pros | ⚠️ Cons |
|---|---|
| Pure ARI — matches `node-asterisk-ari` design exactly | RTP/jitter handling is on you |
| Keep bridge, transfers, recording, playback in ARI | NAT: must set `external_host`/advertise host right |
| Can swap a human bridge for AI bridge with same primitives | Slightly more moving parts than AudioSocket |
| `direction: both` gives full duplex | Needs Asterisk 16+ (we're on 18+) |

> This is the most natural fit for our existing engine: `node-asterisk-ari` already creates bridges and channels via ARI; the AI node just creates an *externalMedia* channel instead of *originating to a human endpoint*.

---

## Option C — Dial to a SIP URI (bridge to LiveKit SIP or any SIP AI)

Don't extract media at all — **route the call onward as SIP** to a service that speaks SIP and does the AI. Most relevant: **LiveKit SIP**, a standalone Go service (Apache-2.0, `livekit/sip`) that bridges SIP ↔ WebRTC and places the caller into a LiveKit **room** where the existing agent serves them.

```asterisk
; hand the call to a PJSIP endpoint pointing at the LiveKit SIP service
same => n,Dial(PJSIP/${EXTEN}@livekit-trunk)
; carry the conversation id as a SIP header (mapped to a participant attribute)
; same => n,Set(PJSIP_HEADER(add,X-Conversation-Id)=${CONV_ID})
```

### How LiveKit SIP actually works (verified)
- **Separate service** coupled to the LiveKit SFU + the **same Redis** and `api_key/secret`. Self-hostable (no per-minute fee) or managed on LiveKit Cloud. Ports: SIP **5060**, RTP **10000–20000/udp**.
- **Inbound trunk** authenticates the peer (digest `auth_username/password` and/or `allowed_addresses` IP allowlist) and can copy SIP headers into participant attributes via **`headers_to_attributes`**.
- **Dispatch rules** choose the room: `dispatchRuleDirect` (fixed name), `dispatchRuleIndividual` (per-caller), `dispatchRuleCallee` (from the dialed number). **Agent dispatch** rides on `roomConfig.agents[]`.
- **Codecs:** SIP leg is **G.711 μ-law/A-law 8 k** (also G.722); LiveKit **transcodes to Opus 48 k automatically** — so *no resampling lands on us*, even with Gemini. DTMF via RFC 2833. Transfer via REFER (`TransferSIPParticipant`); outbound originate via `CreateSIPParticipant` (takes an **explicit `room_name`**).

### Two real gotchas (shape the design — see [[05-implementation-plan]] §3)
> ⚠️ **No room-name templating on inbound.** You cannot write `roomName: "rtc-{X-Conversation-Id}"`. To land in `rtc-{conversationId}` reliably, let LiveKit **originate** the leg with `CreateSIPParticipant({room_name})`, or accept a dispatch-rule room and reconcile `conversationId` (from a header→attribute) inside the agent.

> ⚠️ **chan_pjsip ↔ LiveKit SIP interop is reportedly fiddly** (community favors chan_sip). LiveCaller is PJSIP — validate early; may need transport/`rewrite_contact`/`direct_media=no` tuning.

| ✅ Pros | ⚠️ Cons |
|---|---|
| **Reuses the docker-rtc voice-agent unchanged** (brain + ears) | Adds LiveKit server + SIP service + Redis to the phone path |
| LiveKit handles μ-law↔Opus transcode, jitter, codecs — **no resampling on us** | Room-name can't be templated from a header (use originate path) |
| One AI implementation for web + robot + phone | chan_pjsip interop quirks to validate |
| DTMF (RFC 2833), REFER transfer, agent dispatch <150 ms | SIP trunk auth/routing + an extra media hop (latency) |

---

## Option D — ExternalMedia/AudioSocket → publish into LiveKit room (gateway)

Hybrid of B and C: keep ARI control, but the gateway you stream to **publishes the audio as a LiveKit participant track** into `rtc-{conversationId}` and plays the agent's track back to Asterisk. Reuses the voice-agent like C, but Asterisk/ARI keeps the bridge (so transfer-to-human, recording via ARI, etc. stay native).

| ✅ Pros | ⚠️ Cons |
|---|---|
| Reuses voice-agent **and** keeps ARI control | You build/maintain the gateway (RTP↔LiveKit track) |
| Clean fallback path to a human queue mid-call | Most engineering of all options |
| No SIP trunking between Asterisk and LiveKit | Resampling 8k↔48k in the gateway |

---

## Other mechanisms (rejected for this use case)

- **AGI / FastAGI / EAGI** — request/response dialplan scripting; EAGI streams inbound audio on fd 3 but not bidirectional duplex cleanly. Legacy; not suited to realtime duplex AI.
- **chan_audiosocket as bridge member** — same as Option A essentially.
- **MixMonitor / record then process** — batch only, not conversational.
- **External SIP media server (Janus/FreeSWITCH)** — heavier; we already have LiveKit.

---

---

## Deep dive: AudioSocket vs ExternalMedia RTP

[[03-reference-avr|AVR]] standardized on AudioSocket; [[04-reference-ava|AVA]] standardized on ExternalMedia RTP (with AudioSocket as a remote-split fallback). They solve the same problem at **different layers**, and the choice is not cosmetic.

|                     | **AudioSocket**                                                    | **ExternalMedia (RTP)**                                              |
| ------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| Transport           | **TCP** (one bidirectional connection)                             | **UDP / RTP** datagrams                                              |
| Created by          | dialplan `AudioSocket()` / `Dial(AudioSocket/…)`, or ARI originate | **ARI only**: `POST /channels/externalMedia`                         |
| Framing             | Trivial: `[type:1][len:2 BE][payload]` + UUID/DTMF frames          | Real RTP: 12-byte header, SSRC, seq#, timestamps, payload type       |
| Delivery semantics  | Reliable, ordered (TCP)                                            | Unreliable, unordered (UDP) — **correct for realtime**               |
| Jitter / loss       | None at transport, but **TCP head-of-line blocking**               | You implement a **jitter buffer**; drop late packets                 |
| Media clock         | None — infer timing from 20 ms frame arrival                       | RTP **timestamps** = real clock + gap detection                      |
| DTMF                | Built-in frame type (`0x03`)                                       | RFC 4733 telephone-event, or ARI `ChannelDtmfReceived`               |
| NAT / firewall      | One outbound TCP port — easy                                       | UDP + advertise-host negotiation — finicky                           |
| ARI integration     | Standalone audio pipe; control lives elsewhere                     | **Native** — a real channel in a bridge (snoop/record/transfer free) |
| Implementation cost | Low (any language, no RTP stack)                                   | Higher (RTP parse + jitter buffer)                                   |

### The TCP-vs-UDP-for-realtime tension (the crux)
TCP guarantees every byte arrives in order — but if one packet is delayed, **everything behind it waits** (head-of-line blocking), so a single hiccup becomes a growing latency/jitter bubble. For audio you almost always want the opposite: a late packet is *useless*, so **drop it and play the next one**. RTP's sequence numbers + timestamps exist precisely for that (jitter buffering, loss concealment, A/V sync). So:
- **ExternalMedia RTP = telephony-correct** realtime media semantics.
- **AudioSocket = pragmatically simple** and totally fine on a LAN/datacenter where loss ≈ 0; you trade "proper" media handling for a dead-simple framed pipe.

### The ARI-coupling distinction
ExternalMedia **is a channel object** — once it's in a bridge you inherit Asterisk's whole toolbox (add/remove channels, snoop, record, transfer, Announcer playback). AudioSocket is just a pipe; you manage control separately.

### Why AVR chose AudioSocket
AVR is general-purpose, plug-and-play OSS, and AudioSocket fits that goal:
1. **Zero ARI dependency** — works from a single dialplan line on *any* Asterisk/FreePBX, no Stasis app, no channel/bridge management. Minimal integration surface = adoption.
2. **Trivial in every microservice** — a framed TCP socket needs no RTP stack/jitter buffer; suits AVR's polyglot fleet of small services.
3. **Container/NAT-friendly** — one outbound TCP connection per call maps cleanly to Docker-compose and remote deploys.
4. **UUID handshake = their session model** — AVR keys Core state, webhooks, and per-call STS routing on the AudioSocket UUID.
5. **They don't need ARI control** — AVR's model is "the call **is** the AI conversation"; transfer/hangup are LLM **tools**, not channel swaps. So they lose nothing by skipping ExternalMedia.

Trade-off accepted: TCP's realtime weaknesses + manual timing, worth it for the simplicity.

### Why *we* would lean the other way (ExternalMedia) — **superseded, see below**
`node-asterisk-ari` is already an **ARI bridge engine**, and we want **mid-call escalation to a human queue** (swap a channel in a live bridge — exactly what `QueueAction` does). That makes **ExternalMedia the natural fit for us** — the *opposite* of AVR's calculus. This is why [[05-implementation-plan]] Option D uses ExternalMedia rather than AudioSocket.

> ⚠️ **Revision (Jul 2026, [[12-ava-transport-compatibility-learnings]]):** the paragraph above conflated two axes. "ARI-native" is about **who orchestrates** (Stasis → bridge → channel), not **what frames the wire**. `POST /channels/externalMedia` accepts `encapsulation: "audiosocket"` (Asterisk 18+), and ARI can also originate a `AudioSocket/…` channel into the bridge — either way the media leg is a real bridge channel (escalation/recording/snoop native) with TCP AudioSocket framing on the wire. AVA's field validation (their Transport-Mode-Compatibility doc) shows AudioSocket + full-agent + **streaming** is the battle-tested cell for our STS architecture, while ExternalMedia RTP was only ever validated with pipeline + *file* playback (which needs a shared FS and caused their gating/feedback saga). **Our call: ARI-orchestrated AudioSocket framing primary, ExternalMedia RTP fallback.**

---

## Decision input

- Lowest latency, fully telephony-native, no LiveKit → **Option A or B** + realtime provider (the AVR/AVA pattern).
- Maximum reuse of the AI we already built, one codebase → **Option C (primary) or D (control-preserving)**.

See the matrix and final call in [[06-comparison-and-decision]] and the build steps in [[05-implementation-plan]].
