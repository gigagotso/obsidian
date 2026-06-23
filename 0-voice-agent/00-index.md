# Voice Agent on Asterisk — Research & Implementation

> Goal: add an **"AI Voice Agent" action** to the LiveCaller PBX flow builder, so an inbound (or outbound) SIP/PBX call can be handed to a conversational voice agent — the same kind of agent already running for WebRTC in [[01-current-architecture#docker-rtc (LiveKit + Gemini)|docker-rtc]].

Created: 2026-06-21 · Author: research pass over `livecaller-data/www/api`, `node-asterisk-ari`, `docker-rtc`, plus AVR & AVA open-source projects.

## TL;DR recommendation

We already built a **working AI voice agent** in `docker-rtc`. The crucial distinction (see [[09-standalone-agent-no-livekit]]): the LiveCaller **brain** (assistant config, RAG, tools, transcripts) is **plain HTTP** and reusable over *any* transport; **LiveKit only provides the "ears"** (audio orchestration via `agent.py`). So adding a phone agent reuses the brain regardless — the only question is which **ears**:

1. **Direction 1 — LiveKit SIP** (fastest to validate): route the SIP call into room `rtc-{conversationId}`; the existing tuned agent serves it unchanged. Brain + ears reused, near-zero new code. Cost: extra infra (LiveKit server + SIP), SIP trunking, an extra media hop. [[05-implementation-plan]].
2. **Direction 2 — standalone microservice, NO LiveKit** (leanest end-state, *likely the right long-term telephony home*): Asterisk → ExternalMedia/AudioSocket → thin service → Gemini Live, **reusing the HTTP brain**. The [[03-reference-avr|AVR]]/[[04-reference-ava|AVA]] shape. Lower latency, less infra, native ARI escalation + recording, no SIP trunking. Cost: rebuild the thin ears + re-tune Georgian turn-taking. [[09-standalone-agent-no-livekit]].
3. **Direction 3 — shared brain library + pluggable transport** (best if telephony is strategic): one core, `LiveKitRoomTransport` + `AsteriskMediaTransport` adapters.

**You do NOT need LiveKit for a phone AI agent** — AVR/AVA prove it, and Asterisk already supplies transport + recording. Two research findings sharpen this (see [[06-comparison-and-decision]], [[10-realtime-providers]]): LiveKit SIP **can't template the room name from a header** and **chan_pjsip interop is reportedly fiddly** (we're PJSIP) — friction on the LiveKit path; meanwhile **OpenAI Realtime speaks G.711 μ-law 8 k natively + has SIP + semantic VAD**, which erases the resampling and turn-tuning that dropping LiveKit would otherwise cost. So Direction 2b (no-LiveKit, OpenAI-direct) is now a strong contender, not a fallback.

The **engine (Gemini vs OpenAI) and the transport (LiveKit vs direct) are independent choices** — keep `agent-config` provider-agnostic and all combinations stay open. Suggested play: spike Direction 1 *if quick* to measure latency and PJSIP interop, then build Direction 2 as the real telephony architecture. Full reasoning in [[06-comparison-and-decision]].

## Documents

1. [[01-current-architecture]] — what exists today: PHP API PBX flow, `node-asterisk-ari`, `docker-rtc` (LiveKit/Gemini), the `pbx_flows` builder.
2. [[02-asterisk-audio-integration-options]] — how to get audio out of an Asterisk channel: AudioSocket vs ARI ExternalMedia vs Dial-to-SIP. Protocols, formats, ports.
3. [[03-reference-avr]] — Agent Voice Response (agentvoiceresponse) deep dive.
4. [[04-reference-ava]] — AVA AI Voice Agent for Asterisk (hkjarral) deep dive.
5. [[05-implementation-plan]] — concrete plan: the new flow node, the bridge, conversation wiring, phased rollout.
6. [[06-comparison-and-decision]] — comparison matrix and the decision with rationale.
7. [[07-vad-and-turn-taking]] — VAD (the five jobs), turn detection, Live/Realtime API (STS) vs STT→LLM→TTS pipeline, latency anatomy, barge-in teardown.
8. [[08-noise-cancellation]] — noise/echo handling in the browser: what to implement for the sip.js web phone vs the LiveKit web call (native APM, Krisp, RNNoise, server-side).
9. [[09-standalone-agent-no-livekit]] — the no-LiveKit option: a thin telephony voice-agent microservice reusing the HTTP brain (AVR/AVA shape), brain-vs-ears, and the shared-core middle path.
10. [[10-realtime-providers]] — Gemini Live vs OpenAI Realtime for a *phone* agent: native μ-law vs PCM-only, turn detection, cost, languages, session limits — and why the engine is independent of the transport.
11. [[11-unified-agent-architecture]] — the concrete `docker-rtc` refactor: one codebase, two transports (LiveKit + Asterisk), each call choosing config + flow. Module layout, core interfaces, Mermaid architecture + sequence diagrams, phased migration.

## The core picture

```
                         ┌──────────────────────────────────────┐
   PSTN / SIP trunk ──►   Asterisk (PJSIP)  ──ARI──►  node-asterisk-ari
                         │   pbx_flows flow engine    (flow executor)
                         └──────────────────────────────────────┘
                                        │
                         flow hits the NEW "AI Agent" node
                                        │
              ┌─────────────────────────┴─────────────────────────┐
              ▼ (Recommended)                                      ▼ (Alternative)
   bridge channel into LiveKit room                     ExternalMedia / AudioSocket
   rtc-{conversationId}                                 straight to a realtime AI
              │                                                    │
              ▼                                                    ▼
   EXISTING docker-rtc voice-agent                       new media-gateway → Gemini/
   (Gemini Live, RAG, tools, transcripts)                OpenAI Realtime (AVR/AVA style)
```

The strategic win either way: **one shared brain** (assistant config, RAG, tools, transcripts — all HTTP) serves web widget, robot tablet, **and** phone. LiveKit vs no-LiveKit only changes the **ears** (how audio reaches that brain), not the brain itself — which is why telephony does *not* require LiveKit. See [[09-standalone-agent-no-livekit]].
