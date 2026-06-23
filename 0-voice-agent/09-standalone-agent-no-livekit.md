# 09 — Standalone Voice-Agent Microservice (no LiveKit)

A full evaluation of giving phone calls an AI agent **without LiveKit at all** — a dedicated microservice that connects Asterisk directly to a realtime model, the [[03-reference-avr|AVR]] / [[04-reference-ava|AVA]] shape, **reusing the entire LiveCaller brain over HTTP**. See [[00-index]], the model angle in [[10-realtime-providers]], and the verdict in [[06-comparison-and-decision]].

> This page exists because the original LiveKit-first recommendation over-weighted code reuse. The correction, stated plainly: **the LiveCaller brain is transport-agnostic HTTP — LiveKit is just one set of "ears."** You do not need LiveKit to give a phone call a voice agent, and recent research makes the no-LiveKit path *more* attractive than it first appeared.

---

## Brain vs ears — the distinction everything hinges on

| Layer | What it is | Coupled to LiveKit? |
|---|---|---|
| **Brain** | which assistant answers, what it knows (RAG), what it can do (tools), what it said (transcripts), prompt/voice/model | **No.** Plain HTTP: `agent-config`, `retrieve`, `execute-tool`, `transcript`. |
| **Ears** (audio orchestration) | move audio between the call and the model; run or defer VAD, turn-taking, interruption | **Yes.** `agent.py` is built on the `livekit-agents` SDK, which needs a LiveKit room. |

"Reuse the agent" only ever meant reusing the **ears** — the LiveKit-bound layer. The **brain is reusable over any transport**. So a no-LiveKit telephony path does **not** fork the AI: it calls the same HTTP endpoints and rebuilds only the ears — which, because realtime APIs are speech-to-speech, is a *thin* layer (socket I/O + a model session + interruption glue), not a second AI.

---

## The research finding that changes the math

When this option was first sketched, the implicit model was Gemini, which is **PCM-only** and would force the gateway to resample 8 kHz μ-law ↔ 16/24 kHz PCM on every call. That made "no LiveKit" feel like trading LiveKit's transcoding for your own.

But **OpenAI Realtime speaks telephony natively** — it accepts and emits **G.711 μ-law at 8 kHz directly** and even offers a **SIP transport** ([[10-realtime-providers]]). With OpenAI as the engine, the standalone gateway's hardest job (resampling) largely disappears, *and* you gain **semantic VAD** (better turn-taking for Georgian's irregular pauses) — which also offsets the loss of LiveKit's multilingual turn detector. The trade is cost (OpenAI audio tokens are ~5–6× Gemini's) and running a second provider. Net: **the no-LiveKit path is now genuinely competitive, not a consolation prize.**

So Direction 2 has two flavors:
- **2a — Gemini-direct:** keep one provider across web+phone; gateway owns resampling + (ideally) a ported turn detector.
- **2b — OpenAI-direct:** native μ-law + SIP + semantic VAD; minimal audio plumbing; higher token cost; provider-agnostic `agent-config` needed.

---

## Architecture

```
PSTN / SIP trunk ─► Asterisk (PJSIP)
                       │  ARI: Stasis "incoming"
                       ▼
                 node-asterisk-ari  ── AIAgentAction node ──┐
                       │  POST /channels/externalMedia       │ (or AudioSocket)
                       │  + mixing bridge (caller + extMedia) │
                       ▼                                      │  RTP / TCP audio
        ┌─────────────────────────────────────────────┐     │
        │   telephony-voice-agent  (new microservice)  │◄────┘
        │   ───────────────────────────────────────    │
        │   • receive caller audio (μ-law/slin 8 kHz)   │
        │   • [Gemini] resample 8k↔16k/24k  [OpenAI] none│
        │   • realtime session (VAD+STT+LLM+TTS in model)│
        │   • stream model audio back over the socket   │
        │                                               │
        │   reuses the BRAIN over HTTP (verified shapes):│
        │   ├ GET  /v1/internal/rtc/agent-config/{cid}   │ → assistant_id, system_instruction,
        │   │                                            │   voice, model, (gemini_)api_key, rag, actions[]
        │   ├ POST /v1/internal/rtc/retrieve             │ → { chunks:[{title,content,score}] }
        │   ├ POST /v1/internal/rtc/execute-tool/{cid}   │ ← { assistant_id, action_name, arguments } → { result }
        │   └ POST /webhooks/rtc/transcript              │ ← { room, identity, text, is_final, language }
        └─────────────────────────────────────────────┘
```

**Conversation/assistant wiring is identical to the LiveKit plan** ([[05-implementation-plan]] §1c): attach the assistant via `conversation->aiAssistants()->attach(...)` **and** set `extra_attributes['ai_assistant_id']`, so `agent-config` resolves it and the transcript resolver attributes agent turns. The room name has no special meaning here — but keep `rtc-{conversationId}` as the transcript `room` value so the webhook resolves the session unchanged.

**Escalation to a human (clean, native ARI):** node-asterisk-ari drops the ExternalMedia channel and adds a human PJSIP endpoint to the *same* bridge — exactly what `QueueAction` already does. No SIP-transfer gymnastics.

**Recording (already native):** `node-asterisk-ari/recording.service.js` records the bridge to WAV. No LiveKit egress needed for phone.

---

## What it wins

- **Lower latency** — no SIP↔WebRTC↔SFU hop or extra jitter buffers; Asterisk → gateway → model.
- **Less infra** — no LiveKit server, no LiveKit SIP service, no Redis-for-RTC on the phone path.
- **No SIP trunking to LiveKit** — sidesteps the documented **chan_pjsip ↔ LiveKit SIP interop quirks** and the **room-name templating limitation** entirely ([[05-implementation-plan]] §3, [[06-comparison-and-decision]]).
- **Native ARI fit** — node-asterisk-ari is already a bridge engine; ExternalMedia is one API call, escalation and recording are native.
- **With OpenAI (2b):** native μ-law + SIP transport ⇒ minimal audio plumbing; semantic VAD ⇒ good Georgian turn-taking without porting a detector.
- **Proven shape** — AVR and AVA both do exactly this; neither uses LiveKit.

## What it costs

- **No reuse of `agent.py`'s orchestration** — `AgentSession`, interruption handling, and (for 2a) the **multilingual turn detector** are livekit-agents constructs you'd re-implement. With OpenAI's semantic VAD this is mostly moot; with Gemini-direct you'd lean on bare acoustic VAD or port the detector ([[07-vad-and-turn-taking]]).
- **Re-tuning** — barge-in/greeting/interruption tuned once for the web agent must be re-established on the new service. **Biggest real cost for 2a.**
- **Two audio-orchestration codebases** (web on LiveKit, phone here) unless you adopt the shared-core path below.
- **Provider management** — 2b means a second vendor + a provider-agnostic `agent-config`.
- **Slower first call** than LiveKit SIP, which reuses brain *and* tuned ears immediately.

## Non-differentiators (don't let these decide it)
- **8 kHz quality** — narrowband caps fidelity *regardless* of transport; it is identical for LiveKit and direct. (It only affects *engineering*: who resamples.)
- **The brain** — byte-for-byte the same HTTP reuse either way.

---

## The elegant middle path: shared brain library + pluggable transport

Factor the model glue (realtime session management, tool dispatch, RAG calls, transcript forwarding, prompt building, VAD/turn detector) into a **shared core**, with two thin transport adapters:

```
        ┌──────────────── shared voice-agent core ────────────────┐
        │  realtime session · VAD · turn detector · tools · RAG    │
        │  · transcript forwarding · prompt/greeting building      │
        └───────────────┬────────────────────────────┬────────────┘
                        │                            │
            LiveKitRoomTransport          AsteriskMediaTransport
              (web / robot)                 (phone: ExternalMedia/AudioSocket)
```

One brain, swappable ears; Georgian turn-taking tuned **once** and shared. Upfront refactor cost, but it dissolves the "two codebases" objection and is the best long-term shape if telephony is strategic. **This is now spelled out in full — module layout, core interfaces, diagrams, phased migration — in [[11-unified-agent-architecture]].**

---

## Implementation sketch

1. **node-asterisk-ari** — `AIAgentAction.js` registered in `flowExecutor.service.js`; on the node: ensure conversation, create mixing bridge + `POST /channels/externalMedia` (`format: ulaw` for OpenAI, `slin16` for Gemini; `direction: both`) toward `telephony-voice-agent`, pass `conversationId`. Tear down on `StasisEnd`; escalate via `CallQueueAction`.
2. **telephony-voice-agent** (new, Python recommended):
   - Accept ExternalMedia RTP (or AudioSocket TCP) keyed by `conversationId`.
   - `GET /v1/internal/rtc/agent-config/{conversationId}` → assistant config.
   - Open the realtime session (OpenAI Realtime *or* `google-genai` live); wire tools → `execute-tool`, RAG → `retrieve`.
   - **[Gemini only]** resample 8k↔16k(in)/24k(out); **[OpenAI]** pass μ-law through.
   - Forward transcripts → `/webhooks/rtc/transcript` with identities `ai_assistant-{assistantId}` / `visitor-{conversationId}`.
   - Add a telephony `transfer`/`hangup` tool.
   - Handle session limits (Gemini ~15 min ⇒ session resumption; see [[10-realtime-providers]]).
3. **PHP API** — assistant attach (§1c of [[05-implementation-plan]]); make `agent-config` provider-agnostic if adopting OpenAI.
4. **Frontend** — the same `AIAgentAction` node ([[05-implementation-plan]] §5); transport is invisible to the builder.

Parts 1, 2, 5 of [[05-implementation-plan]] are essentially **identical**; only the bridge (Part 3) changes — and here there is no LiveKit at all.

---

## When to pick which

| Pick **C / Direction 2 (no LiveKit)** if… | Pick **A / Direction 1 (LiveKit SIP)** if… |
|---|---|
| telephony is strategic / high-volume | you want a working phone agent in days |
| latency & lean infra matter most | you won't re-tune turn-taking now |
| you'd rather avoid SIP-trunking-to-LiveKit + its quirks | multi-party / supervisor-listen is needed |
| you're open to OpenAI for telephony (native μ-law) | one audio codebase across all channels is a mandate |

**Suggested sequence:** spike Direction 1 *if it's quick* to get a real latency number with zero new orchestration → build Direction 2 (likely **2b, OpenAI-direct**, or the shared-core middle path) as the real telephony architecture once validated. Full reasoning in [[06-comparison-and-decision]].
