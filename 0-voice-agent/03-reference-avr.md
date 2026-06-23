# 03 — Reference: Agent Voice Response (AVR)

Open-source (MIT), microservices "voice AI layer for Asterisk." Org: https://github.com/agentvoiceresponse · Wiki: https://wiki.agentvoiceresponse.com/ . The cleanest reference for the **AudioSocket + swappable-providers** pattern. See [[00-index]], compare in [[06-comparison-and-decision]].

## Architecture

`avr-core` is the orchestrator; every ASR/LLM/TTS/STS provider is its own small Node/Express microservice on a shared Docker network. `avr-infra` ships ~16 ready `docker-compose-*.yml` (one per provider combo) + a bundled `avr-asterisk` image.

| Component | Role |
|---|---|
| **avr-core** | AudioSocket TCP server toward Asterisk; HTTP/WS clients toward AI; runs VAD, barge-in, tools, webhooks |
| **avr-infra** | Compose files, `.env`, Asterisk config |
| **ASR** | `avr-asr-deepgram/-google/-vosk/-soniox/-sarvam` |
| **LLM** | `avr-llm-openai/-anthropic/-openrouter/-n8n` |
| **TTS** | `avr-tts-elevenlabs/-google/-cartesia/-kokoro/-coquitts` |
| **STS** | `avr-sts-openai/-gemini/-ultravox/-elevenlabs/-deepgram/...` (speech-to-speech, one socket) |
| **avr-vad / avr-app** | Silero VAD lib · agent design dashboard |

**Two pipelines:** Classic **ASR→LLM→TTS** (3 services, full control + tools) or **STS** (caller audio streamed to one realtime provider, e.g. OpenAI Realtime / Gemini Live — lower latency, provider owns VAD).

## Asterisk integration — AudioSocket

Native AudioSocket (Asterisk 18+), **not** ARI/AGI. Core runs a TCP server (default **:5001**); Asterisk dials out to it.

```asterisk
[demo]
exten => 5001,1,Answer()
 same => n,Set(UUID=${SHELL(uuidgen | tr -d '\n')})
 same => n,AudioSocket(${UUID},AVR_HOST:5001)   ; or Dial(AudioSocket/host:5001/${UUID})
 same => n,Hangup()
```
Protocol = the standard AudioSocket framing (see [[02-asterisk-audio-integration-options#Option A — AudioSocket]]). The **UUID per call** ties the audio stream ↔ Core state ↔ webhook/STS metadata.

## Audio flow

Asterisk 20 ms frames → Core auto-detects codec (μ-law/A-law/slin via RMS) → standardizes to **8 kHz/16-bit mono PCM, 320-byte (20 ms) chunks** → streams to ASR (HTTP streamed) → final transcript → LLM → text streamed into TTS → TTS returns 8 kHz PCM → Core writes frames back over the AudioSocket. Internal format throughout = **slin16**.

## Core orchestration

Node app, two servers: AudioSocket TCP (`PORT=5001`) + optional HTTP (`HTTP_PORT`, e.g. 6001: `GET /health`, `POST /call` for call-init metadata, can return a per-call `sts_url`). Service wiring via env:
```
ASR_URL=http://avr-asr-*:6010/speech-to-text-stream
LLM_URL=http://avr-llm-*:6002/prompt-stream
TTS_URL=http://avr-tts-*:6012/text-to-speech-stream
# OR mutually exclusive:
STS_URL=ws://avr-sts-*:6033
```
(Exact paths/ports are per-provider conventions set in each compose file — treat as configurable.)

## VAD / barge-in

Silero VAD in Core, env-tuned for telephony (aggressively low thresholds). Key switch `INTERRUPT_LISTENING`: `false` (default) keeps ASR live during TTS → caller can barge-in; `true` disables it. Tunables: `VAD_POSITIVE_SPEECH_THRESHOLD` (0.08), `VAD_NEGATIVE_SPEECH_THRESHOLD` (0.03), `VAD_MIN_SPEECH_FRAMES` (3), `VAD_REDEMPTION_FRAMES` (8), etc. `AMBIENT_NOISE_FILE`/`_LEVEL` blend low background noise to cut dead air. STS providers do their own VAD.

## Function calling / tools

Auto-loaded from `avr_tools/` (built-in) + `tools/` (user). Each tool = a Node module:
```js
module.exports = {
  name: "get_weather",
  description: "Retrieves current weather...",
  input_schema: { type:"object", properties:{ location:{type:"string"} }, required:["location"] },
  handler: async (uuid, { location }) => `The weather in ${location} is sunny.` // string spoken back
};
```
Built-ins: `avr_transfer` (to an extension), `avr_hangup`. Plus MCP server + n8n-as-LLM-brain integrations.

## Webhooks

`POST WEBHOOK_URL`, envelope `{uuid,type,timestamp,payload}`. Events: `call_initiated` (can override routing with `sts_url`), `call_started`, `transcription`, `interruption`, `dtmf_digit` (Asterisk 22+), `call_ended`. Auth via `WEBHOOK_SECRET` (`X-AVR-WEBHOOK-SECRET`), retry/backoff configurable.

## What we borrow

- The **AudioSocket framing + 8 kHz slin16 / 20 ms chunk** discipline → if we go [[02-asterisk-audio-integration-options#Option A — AudioSocket|Option A]].
- The **STS-vs-pipeline** distinction — our docker-rtc agent is already STS (Gemini native audio), so we lean STS.
- The **tool module contract** maps 1:1 onto our existing `/v1/internal/rtc/execute-tool`.
- **Per-call UUID = conversationId** convention.
- VAD/barge-in tunables as defaults if we ever run our own VAD (we don't need to with Gemini Live, which has turn detection built in).
