# 10 — Realtime Voice Providers: Gemini Live vs OpenAI Realtime

The model is the engine of the voice agent — it decides latency, cost, language quality, and (crucially for telephony) **whether you have to resample audio at all**. This page compares the two production-grade speech-to-speech APIs as they bear on a *phone* agent. See [[00-index]], the VAD/turn-taking context in [[07-vad-and-turn-taking]], and how the choice lands in [[09-standalone-agent-no-livekit]].

> Figures below were gathered June 2026 and move fast — treat model IDs and prices as "verify before you commit," not gospel.

---

## The single most important fact for telephony

A phone call is **G.711 μ-law (or A-law), 8 kHz, mono**. What each model accepts at its input boundary therefore decides how much audio plumbing you own:

- **OpenAI Realtime speaks telephony natively.** It accepts and emits **`audio/pcmu` (G.711 μ-law) and `audio/g711_alaw` at 8 kHz directly**, *and* it has a first-class **SIP transport** (`/v1/realtime/calls`). For a phone agent this is enormous: the caller's bytes can flow to the model essentially untouched — no resampling, no format juggling.
- **Gemini Live is PCM-only.** Input must be **16 kHz 16-bit little-endian PCM**; output is **fixed at 24 kHz PCM**. There is no μ-law I/O. So a Gemini telephony path *must* resample **8 kHz μ-law → 16 kHz PCM inbound** and **24 kHz PCM → 8 kHz μ-law outbound**, on every call, in your gateway (or let LiveKit do it).

This one difference quietly reframes the whole telephony decision: **OpenAI Realtime is the lower-friction engine for a no-LiveKit phone agent**, while **Gemini is what LiveCaller already runs for the web** (where the browser delivers wideband audio and resampling is a non-issue). It's a genuine pull in two directions — see the recommendation at the bottom.

---

## Side-by-side

| Dimension | **Gemini Live** | **OpenAI Realtime** |
|---|---|---|
| Telephony audio | ❌ PCM only — **16 k in / 24 k out**, must resample | ✅ **native μ-law/A-law 8 k**, plus PCM16 |
| Transport | WebSocket only (`BidiGenerateContent`) | **WebRTC, WebSocket, *and* SIP** |
| Turn detection | acoustic VAD (start/end sensitivity, `silenceDurationMs`≈800) | **`server_vad`** *and* **`semantic_vad`** (LLM judges completion, `eagerness` low→high) |
| Interruption | server sets `serverContent.interrupted:true`; you stop playback | auto-cancel on speech; you send `response.cancel` + **`conversation.item.truncate`** |
| Transcription side-channel | `inputAudioTranscription`/`outputAudioTranscription` | `input_audio_transcription` + `response.output_audio_transcript.delta` |
| Function calling | tools in setup; 2.5 supports **async/non-blocking** tools; native-audio model reportedly **weaker at tool calling** | tools + **remote MCP**; `function_call` in `response.done`, reply with `function_call_output` |
| Session limits | **audio ~15 min**, WS conn ~10 min; needs **session resumption** / context compression | governed by 32k context window (historical ~30 min cap — verify) |
| Latency (TTFA) | "sub-second" (no precise official number) | **~500 ms TTFB**, ~250–500 ms end-to-end |
| Audio token price (per 1M) | **in $3 / out $12** | **in $32 / out $64** (cached in $0.40) |
| Languages | ~70 spoken; **24 langs / 30 HD voices**, auto-switch mid-call | multilingual; **semantic VAD helps disfluent/low-resource speech** |
| Architecture | **native-audio** *or* **half-cascade** (audio→text→TTS) | pure native S2S |
| Standalone (no LiveKit)? | ✅ `google-genai` `client.aio.live.connect` | ✅ `openai` SDK (WebRTC/WS) or raw WS/SIP |

---

## Reading the table: what actually matters

**Cost.** Audio tokens dominate a voice agent's bill because you pay for *every second in both directions*, silence included until the VAD trims it. Gemini's native-audio rates are **~5–6× cheaper** than OpenAI's on raw audio. At OpenAI's prices, spoken *output* alone runs on the order of ~$0.06/min; Gemini is a fraction of that. At PBX scale this is not a rounding error — it can be the deciding factor, and it argues for tight `silence_duration_ms`, concise responses, and prompt caching whatever you pick.

**Turn-taking for Georgian (and other non-fluent/low-resource speech).** This is the quality risk that matters most for LiveCaller. OpenAI's **semantic VAD** judges *whether the thought is finished* from content, so it tolerates the longer, irregular pauses typical of non-native or lesser-resourced speech far better than a pure silence timer. Gemini offers only acoustic VAD — which is exactly *why* the LiveCaller web agent layers LiveKit's `MultilingualModel` turn detector on top of it ([[07-vad-and-turn-taking]]). If a telephony path drops that LiveKit layer and leans on Gemini's bare acoustic VAD, expect more mid-sentence cutoffs in Georgian. Mitigations: keep using a semantic turn detector, or use OpenAI's semantic VAD, or port the multilingual detector into the gateway.

**Tool calling.** LiveCaller's agent is tool-driven (the dynamic `execute-tool` actions). Community reports flag Gemini's **native-audio** model as noticeably *weaker* at tool calling than its **half-cascade** sibling (`gemini-live-2.5-flash-preview`) or OpenAI. If reliability of tools/functions becomes a problem, the half-cascade Gemini path (audio in → text reasoning → separate TTS) trades a little vocal naturalness for markedly better tool reliability and easier logging.

**Native-audio vs half-cascade (Gemini only).** *Native-audio* generates speech directly from the model's internal state — best prosody, lowest latency, affective/“proactive” audio. *Half-cascade* reasons in text then uses a separate TTS — less natural voice, but more reliable tools and more robust in production. OpenAI Realtime is pure native S2S; its transcription is a parallel side-channel, never in the audio path.

**Session length.** Gemini's ~15-min audio session / ~10-min WS connection means **long calls require session-resumption handles or context-window compression** — real engineering for a phone agent where 20-minute calls happen. OpenAI's limit is context-bound (verify the current wall-clock cap). Either way, plan for graceful mid-call reconnection.

---

## How this maps onto LiveCaller today

The existing `agent-config` endpoint (`RTCAgentConfigController`) returns a **Gemini-shaped** payload — note the literal field **`gemini_api_key`** and `model` drawn from `integration.model.model_name` (e.g. `gemini-2.5-flash-native-audio-preview-12-2025`). The web agent (`agent.py`) consumes exactly that. So:

- **Reusing the web stack as-is = Gemini.** Any LiveKit-based phone path ([[05-implementation-plan]] Direction 1) inherits Gemini and its resampling need — but **LiveKit does the μ-law↔PCM bridging for you**, so the resampling burden disappears into the SIP service. This is a real, underappreciated argument *for* LiveKit when staying on Gemini.
- **A no-LiveKit phone path ([[09-standalone-agent-no-livekit]]) forces a provider decision.** If you keep Gemini, your gateway owns the 8 k↔16 k/24 k resampling. If you switch telephony to **OpenAI Realtime**, you get native μ-law + SIP transport + semantic VAD — at higher token cost and with a second provider to manage. To support OpenAI, `agent-config` would need to become provider-agnostic (rename/extend beyond `gemini_api_key`, surface provider + model + key generically). The `AiIntegration`/`AiAction`/`VectorStore` model already abstracts the provider, so this is a modest change, not a rebuild.

---

## Recommendation

- **Web / robot:** stay on **Gemini native-audio** — wideband input, cheapest audio tokens, strong multilingual, and the multilingual turn detector already compensates for acoustic-only VAD.
- **Phone, if you keep LiveKit:** stay on **Gemini** and let **LiveKit SIP** absorb the μ-law↔PCM transcoding and provide the tuned turn-taking. Lowest *new* effort.
- **Phone, if you go no-LiveKit:** seriously evaluate **OpenAI Realtime** — native μ-law 8 k, SIP transport, and semantic VAD make it the natural telephony engine, and they erase the resampling + turn-tuning work that dropping LiveKit would otherwise create. Price is the trade-off; benchmark Georgian quality on both before deciding.

The deeper point: **the provider is not bolted to the transport.** Gemini-via-LiveKit, Gemini-direct, and OpenAI-direct are all viable; the right pairing depends on whether you're optimizing for *least new code* (Gemini + LiveKit), *lowest cost* (Gemini), or *least telephony plumbing* (OpenAI direct). Keep the `agent-config` contract provider-agnostic and you keep all three doors open.

### Verify before committing
- Current OpenAI Realtime wall-clock session cap and `gpt-realtime-mini` pricing/IDs.
- Gemini 3.x live model IDs (LiveKit docs hint at `gemini-3.1-flash-live-preview`; confirm on `ai.google.dev/gemini-api/docs/models`).
- Official Gemini latency numbers (only "sub-second" marketing found).
- Per-language turn-taking quality for Georgian — **neither vendor publishes this; test empirically.**
