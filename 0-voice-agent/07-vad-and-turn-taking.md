# 07 — VAD, Turn-Taking & Live-API vs Pipeline

Why our `docker-rtc` agent runs **Silero VAD + a multilingual turn detector even though Gemini Live has its own**, what VAD actually does, and the full trade-off between **Live/Realtime APIs (speech-to-speech)** and a **classic STT→LLM→TTS pipeline**. See [[00-index]], [[01-current-architecture]], [[08-noise-cancellation]], references [[03-reference-avr]] / [[04-reference-ava]].

---

## What VAD actually is, and the five jobs it does

VAD = **Voice Activity Detection**: a classifier over tiny audio frames (10–30 ms) that outputs speech / not-speech (Silero gives a 0–1 probability). **WebRTC VAD** = old GMM-based, fast but crude (AVA uses it); **Silero** = small neural VAD, robust to noise (we + AVR use it).

That one signal is reused for **five distinct jobs** — conflating them is where confusion starts:

1. **Segmentation** — chop the continuous mic stream into utterances.
2. **Endpointing** — VAD + a *silence timer* = "user stopped → turn is over."
3. **Barge-in detection** — "user started talking while the agent is speaking."
4. **Gating** — only send audio to (expensive) STT when speech is present.
5. **Pre-speech padding** — keep ~200 ms *before* detected onset so word starts aren't clipped.

### The endpointing-timeout tension (the key lever)
The single most important VAD parameter is the **end-of-turn silence timeout**:
- Short (~200 ms) → snappy, but cuts people off when they pause to think.
- Long (~800 ms) → patient, but every turn carries ~800 ms of dead air before the reply.

Other knobs: positive/negative thresholds (hysteresis), min-speech duration (filter clicks/coughs), redemption/hangover frames, pre-speech pad. (AVR exposes all of these as env vars — see [[03-reference-avr#VAD / barge-in]].)

### VAD ≠ turn detection
Pure VAD endpointing only knows *acoustic silence*. A **turn detector** (LiveKit `MultilingualModel`, or a realtime API's internal one) uses **content/prosody** to predict whether the human is actually *done*:
- "I'd like to book a flight to…" → not done, even after 800 ms silence.
- "Book me a flight to Paris." → done after ~250 ms.

It lets you keep a short base timeout but *extend* it when more is likely coming. That's the difference between a bot that talks over you and one that feels natural.

---

## Paradigm A — Live / Realtime APIs (Speech-to-Speech, "STS")

Gemini Live, OpenAI Realtime, etc. **One multimodal model**, audio in → audio out, over a persistent WebSocket/gRPC. Perception, reasoning, and speech generation happen inside one neural system.

```
        ┌──────────────────────────────────────────┐
audio ─► │  Gemini Live / OpenAI Realtime            │ ─► audio
        │  (VAD + turn-taking + reasoning + voice)  │
        └──────────────────────────────────────────┘
         one streaming pass · ~300–800 ms to first audio
```

**Gives you:**
- **Turn-taking built in, server-side** — stream continuous audio; the API emits `speech_started` / `speech_stopped` / turn-complete and decides when to answer.
- **Native interruption** — stream user audio while it talks → it cancels its output and emits an interrupt event. One event tears the response down.
- **Paralinguistics survive** — the model *hears* tone, emotion, hesitation, accent, laughter, and *generates* expressive speech directly (no text round-trip).
- **Lowest latency** — no serialize-to-text-then-resynthesize chain.

**Costs you:**
- **Less observability/control** — no clean "user said X → LLM decided Y" boundary by default. You recover it via the side-channel `input_audio_transcription` / `output_audio_transcription` (exactly why our `agent.py` enables them).
- **Provider lock-in** on voices, languages, model, turn-taking tuning.
- **Cost** = continuous audio token streaming (in + out), often pricier than text, harder to gate.
- **Limited turn-taking knobs** — take the provider's tuning or override it (we override).
- **Hard to inject mid-pipeline logic** (PII redaction before the LLM, custom RAG timing, per-sentence voice switching).

**Where VAD still fits in STS:** the provider has its own VAD, but you *also* run a local one to (a) override turn detection for better/multilingual behavior, (b) catch barge-in locally to stop playback instantly, (c) drive active-speaker UI. That is our Silero + `MultilingualModel` layer on top of Gemini.

### Why our agent layers Silero + turn detector on top of Gemini Live
Confirmed in `docker-rtc/voice-agent/agent.py`:
```python
session = AgentSession(
    llm=google.realtime.RealtimeModel(**model_kwargs),  # Gemini Live (STS)
    vad=silero.VAD.load(),                              # our own VAD
    turn_handling=TurnHandlingOptions(
        turn_detection=MultilingualModel(),            # our own turn detector
    ),
)
# file comment: turn detector "works on top of Silero VAD silence signals"
#               and "prevents cutting off mid-sentence (better than raw VAD)"
```
Reasons: (1) **quality/control** — context-aware, multilingual endpointing prevents mid-sentence cutoffs for Georgian/Russian/code-switched callers; (2) **barge-in & UI** — a local VAD gives instant signals without a model round-trip; (3) **provider independence** — the turn layer is identical regardless of the model behind it.

---

## Paradigm B — Pipeline STT → LLM → TTS (cascade)

Three independent, swappable components:

```
audio ─► [STT/ASR] ─► text ─► [LLM] ─► text ─► [TTS] ─► audio
          (Deepgram)        (GPT/Claude)      (ElevenLabs)
   you own: endpointing · barge-in · stage hand-off timing
```

### Latency is additive — the core weakness
```
end-of-turn silence wait   200–700 ms
+ STT finalization          50–200 ms
+ LLM time-to-first-token  200–600 ms
+ TTS time-to-first-audio  100–300 ms
= ~0.8–1.8 s if streamed well; 2–4 s if naive
```
Mitigations: stream every stage; **chunk TTS at sentence boundaries** (start speaking before the LLM finishes); overlap stages; filler audio ("one moment…").

**Gains:** maximum control & observability (clean transcript per stage; inject guardrails/RAG/redaction/analytics *between* stages); best-of-breed per stage (an STT tuned for 8 kHz telephony, your LLM, your TTS voice); mature function-calling via the text LLM; cost control (gate STT with VAD, cheaper models per stage); no single-provider lock-in; can run fully local/offline.

**Loses:** higher latency (cascade); **paralinguistics gone** (STT discards tone; TTS resynthesizes from flat text — clawed back partially with SSML/emotion tags); **you own turn-taking** — real work.

### Do we still need Silero VAD in a pipeline? Yes.
| Need | Can come from | Local VAD required? |
|---|---|---|
| **Endpointing** ("user done → send to LLM") | STT built-in endpointing (Deepgram `endpointing`/utterance-end, Google endpointer) **or** Silero + silence timer | No — STT can cover it |
| **Barge-in** ("user spoke during TTS → stop agent") | **Local fast VAD** during playback | **Yes** — the part you don't get free |
| **Gating** ("don't run STT on silence") | Local VAD | Optional (cost) |

Endpointing can lean on a good streaming STT; **barge-in needs a local VAD** because during TTS playback the STT stream is paused/lagged. Quality caveat: raw silence-timeout endpointing cuts people off — reuse the same `MultilingualModel` turn detector for good multilingual turns (it runs on transcript + VAD, independent of the model).

---

## The dimension that differs most: interruption (barge-in)

**STS:** user speaks → provider detects → emits interrupt → model stops → you stop playback. ~one event. Edge case is *false* interruptions (background noise, backchannels like "mm-hmm") — handled by the model + turn-detector confidence.

**Pipeline:** your **local VAD** fires (STT is paused/lagged during playback), then you orchestrate a teardown:
1. Stop & flush TTS audio already buffered to the caller.
2. Stop the TTS renderer.
3. Cancel the in-flight streaming LLM response (or finish-and-discard).
4. Restart STT for the new utterance.
5. **Decide if it was real** — backchannel/cough filtering via energy threshold + min-duration + cooldown (AVA's `barge_in.min_ms`/`cooldown_ms`; AVR's `INTERRUPT_LISTENING` keeps a local VAD alive during playback).

That five-step teardown is the single biggest reason a naive pipeline *feels* worse than an STS API — and the single biggest reason you keep a **local** VAD in a pipeline even when STT does endpointing.

---

## Telephony-specific stakes (for this project)

- **8 kHz narrowband:** STS models are trained mostly on wideband; upsampled 8 kHz loses info and can hurt understanding. A pipeline lets you pick an STT *explicitly tuned for telephony* (e.g. Deepgram telephony). **Strongest argument for pipeline on phone.**
- **Latency intolerance:** phone callers have no visual feedback, so dead air feels worse → favors STS.
- **Noise:** street/car/crowd → neural VAD (Silero) + denoise matter more (see [[08-noise-cancellation]]).
- **Barge-in is constant** on phone → robust local barge-in is non-negotiable.

---

## The architectural payoff: swappable model behind a fixed turn layer

```
                 ┌───────────────────────────────────────────┐
 caller audio ─► │ Silero VAD  →  MultilingualModel turn det. │  ── stays the same ──►
                 └───────────────────────────────────────────┘
                                     │  swap only this:
        ┌────────────────────────────▼─────────────────────────────┐
        │  llm = google.realtime.RealtimeModel(...)   # STS (today)  │
        │  ── OR ──                                                   │
        │  stt=Deepgram ; llm=GPT/Claude ; tts=ElevenLabs (pipeline) │
        └────────────────────────────────────────────────────────────┘
```
Only the `llm=`/`stt=`/`tts=` lines change; the VAD + turn detector + barge-in + RTC wiring (RAG, tools, transcripts) stay put. Keeping our own VAD/turn layer now future-proofs the model choice and keeps telephony and web on identical turn behavior.

## STS vs Pipeline — summary

| | **STS (Live API)** | **Pipeline (STT→LLM→TTS)** |
|---|---|---|
| Turn-taking | built-in (we override) | **you build it** |
| Local VAD needed? | optional (barge-in/UI/override) | **yes — esp. barge-in** |
| Latency | lowest | additive; needs streaming tricks |
| Interruption | ~one event | 5-step teardown + false-trigger filter |
| Paralinguistics | preserved | lost (resynth from text) |
| Control / observability | low (transcripts via side-channel) | high (clean per-stage) |
| 8 kHz telephony | model upsamples; some loss | pick a telephony-tuned STT ✅ |
| Cost | continuous audio tokens | gate/swap per stage |
| Function calling | provider-dependent | LLM-stage native (easy) |
| Lock-in | high | low (best-of-breed/local) |

**Net for livecaller:** STS (Gemini Live) is the right default — best latency, native barge-in, preserves tone — with our own Silero + multilingual turn detector on top for controllable, multilingual turn-taking and instant barge-in. The pipeline is the escape hatch if 8 kHz quality, cost, control, or lock-in become the binding constraint, and switching is mostly a `llm=`/`stt=`/`tts=` change.
