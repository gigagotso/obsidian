# 06 — Comparison & Decision

Final comparison of the two strategic directions and the bridge mechanisms. See [[00-index]], [[02-asterisk-audio-integration-options]], [[05-implementation-plan]].

---

## Correction up front: brain vs ears

An earlier draft framed this as "reuse LiveKit" vs "build a second AI stack." That was **wrong**. The LiveCaller **brain** (assistant config, RAG, tools, transcripts) is **plain HTTP** and reusable over *any* transport. LiveKit only provides the **ears** (audio orchestration via `agent.py`/`livekit-agents`). So a no-LiveKit telephony path does **not** fork the AI — it reuses the brain over HTTP and rebuilds only the (thin, with STS) ears. See [[09-standalone-agent-no-livekit]].

## Three directions

### Direction 1 — Bridge into LiveKit, reuse `agent.py` (fastest to validate)
PBX call → LiveKit room → existing tuned agent serves it. Reuses **brain + ears** with near-zero new code. ([[05-implementation-plan]].)

### Direction 2 — Standalone telephony microservice, **no LiveKit** (leanest end-state)
Asterisk → ExternalMedia/AudioSocket → new thin service → Gemini Live, **reusing the HTTP brain**. AVR/AVA shape. Lower latency, less infra, native ARI escalation + recording. Rebuilds only the ears. ([[09-standalone-agent-no-livekit]].)

### Direction 3 — Shared brain library + pluggable transport (best long-term)
Refactor the model glue into a shared core; thin `LiveKitRoomTransport` (web/robot) + `AsteriskMediaTransport` (phone) adapters. One brain, swappable ears, Georgian tuning shared. Upfront refactor. Fully specified — module layout, core interfaces, diagrams, phased migration — in **[[11-unified-agent-architecture]]**.

---

## Matrix

| Criterion | Dir 1: LiveKit + `agent.py` | Dir 2: standalone, no LiveKit | Dir 3: shared core |
|---|---|---|---|
| **Brain reuse** (config/RAG/tools/transcripts) | ✅ HTTP | ✅ **HTTP (same)** | ✅ HTTP |
| **Ears reuse** (audio orchestration) | ✅ `agent.py` as-is | ❌ rebuild (thin w/ STS) | ✅ shared lib |
| **Turn-taking tuning (KA/RU)** | ✅ already tuned | ⚠️ re-tune or rely on Gemini | ✅ shared, tuned once |
| **Latency (phone)** | ⚠️ extra SIP↔WebRTC↔SFU hop | ✅ lowest (direct) | ✅ lowest (direct adapter) |
| **8 kHz handling** | upsampled — *same as Dir 2* | upsampled — *same as Dir 1* | same |
| **Ops footprint** | + LiveKit server + LiveKit SIP + Redis | only the thin service | only the thin service |
| **SIP trunking Asterisk↔LiveKit** | ⚠️ required (fiddly) | ✅ none | ✅ none |
| **Mid-call transfer to human** | OK (SIP transfer) / native (gateway) | ✅ native ARI (swap in bridge) | ✅ native ARI |
| **Recording** | LiveKit egress | ✅ native node-asterisk-ari | ✅ native |
| **Multi-party / supervisor-listen** | ✅ room model | ❌ harder (1:1) | ✅ via LiveKit adapter |
| **Codebases for voice** | 1 (shared w/ web) | 2 (web + phone) | 1 (shared core) |
| **Time-to-value** | ✅ days | weeks | weeks+ (refactor) |

---

## Bridge mechanisms (within Direction 1)

| | Option C: LiveKit SIP | Option D: ExternalMedia → LiveKit gateway |
|---|---|---|
| New code | Least (config + dispatch) | A media gateway (RTP↔LiveKit track, resampling) |
| ARI keeps call control | ❌ handed to LiveKit on dial | ✅ bridge stays in `node-asterisk-ari` |
| Escalation to human | via SIP transfer | ✅ native ARI (swap channel in bridge) |
| Best for | fast time-to-value, AI-only calls | escalation-heavy flows, max control |

And if Direction 2 is ever chosen, pick between [[02-asterisk-audio-integration-options#Option A — AudioSocket|AudioSocket]] (simple TCP, AVR-style) and [[02-asterisk-audio-integration-options#Option B — ARI ExternalMedia|ARI ExternalMedia]] (AVA-style, fits our ARI engine better).

---

## Verified specifics that shape the decision

Two research findings moved the needle since the first draft:

**1. LiveKit SIP has friction we have to design around** ([[02-asterisk-audio-integration-options#Option C — Dial to a SIP URI (bridge to LiveKit SIP or any SIP AI)]]):
- **No room-name templating on inbound** — `rtc-{conversationId}` can't come from a header; you must let LiveKit *originate* the leg (`CreateSIPParticipant({room_name})`) or reconcile in the agent.
- **chan_pjsip ↔ LiveKit SIP interop is reportedly fiddly** (community favors chan_sip) — and **LiveCaller is PJSIP**. This is a concrete, must-validate risk for Direction 1.
- Upside confirmed: LiveKit **transcodes μ-law 8 k ↔ Opus 48 k itself**, so staying on Gemini via LiveKit means **no resampling on us**.

**2. The provider choice is now a real lever** ([[10-realtime-providers]]):
- **OpenAI Realtime speaks G.711 μ-law 8 k natively and has a SIP transport** + **semantic VAD** (better for Georgian's irregular pauses). This makes **Direction 2b (no-LiveKit, OpenAI-direct)** much stronger than first thought: it erases both the resampling burden *and* the turn-detector-porting burden that dropping LiveKit would otherwise create.
- **Gemini is PCM-only** (16 k in / 24 k out) and **~5–6× cheaper** on audio tokens, but acoustic-VAD-only and weaker at tool calling on native-audio. Gemini-direct (2a) therefore owns resampling + turn-tuning; Gemini-via-LiveKit (Dir 1) hides both inside LiveKit.

Net: the engine and the transport are **independent choices**. Gemini+LiveKit (least new code), Gemini-direct (cheapest), and OpenAI-direct (least telephony plumbing) are all coherent — keep `agent-config` provider-agnostic to hold all three open.

---

## Decision (revised)

The brain is shared no matter what (HTTP), so the choice is purely about the **ears**: reuse LiveKit's, or build a thin telephony set. Reframed as **time-to-value vs. optimized end-state**:

1. **Validate with Direction 1 (LiveKit SIP) — *if* it's a quick spike.** It reuses both brain and already-tuned ears with near-zero new code, so you learn real phone latency/quality fast.
2. **Build Direction 2 (standalone, no LiveKit) as the real telephony architecture.** Leaner, lower-latency, telephony-native, reuses the entire HTTP brain, fits node-asterisk-ari (native escalation + recording), no SIP trunking. This is very likely the better long-term home for phone — the original LiveKit-permanent recommendation over-weighted ear reuse. See [[09-standalone-agent-no-livekit]].
3. **Choose Direction 3 (shared core) if** telephony is strategic and you want to stop maintaining two voice codebases — refactor the glue into a shared library with pluggable transports.

**Pick LiveKit-permanent (stay on Direction 1) only if** multi-party/supervisor-listen is required, or a single-audio-codebase mandate outweighs the infra/latency cost.

**The risks to retire first (spike before committing):**
- **Latency** — measure SIP↔LiveKit↔Gemini (Dir 1) *vs* direct Asterisk→gateway→model (Dir 2). Phone callers are latency-intolerant.
- **chan_pjsip ↔ LiveKit SIP interop** — does our PJSIP stack dial LiveKit SIP cleanly, or do we need chan_sip/tuning? (Direction 1 only — and a reason Direction 2 looks safer.)
- **Georgian turn-taking** — does the chosen engine hold up for KA/RU? Gemini-direct needs the ported detector; **OpenAI's semantic VAD may solve this outright** ([[10-realtime-providers]]). This — not 8 kHz, which is identical either way — is the real quality risk.
- **Cost vs plumbing** — Gemini is ~5–6× cheaper per audio token but needs resampling/tuning; OpenAI is pricier but telephony-native. Decide which constraint binds.
