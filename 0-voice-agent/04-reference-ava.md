# 04 — Reference: AVA (AI Voice Agent for Asterisk)

Open-source (MIT), `github.com/hkjarral/AVA-AI-Voice-Agent-for-Asterisk`. ~1.1k★, v7.x. Python 63% / TypeScript 30% (admin UI) / Go. The cleanest reference for the **ARI-orchestrated ExternalMedia** pattern. See [[00-index]], compare in [[06-comparison-and-decision]].

## Architecture

Two-container runtime via Docker Compose (`asterisk-ai-voice-agent`):
- **`ai_engine`** (Python 3.11+) — core. ARI client (events over WS, commands over REST), audio transport servers (AudioSocket TCP / RTP), STT→LLM→TTS pipeline or full-agent providers. Health/metrics `:15000`.
- **`local_ai_server`** (optional, `:5002`) — local STT/LLM/TTS on GPU.
- **`admin_ui`** (Node/TS, `:3003`) — setup wizard, agent CRUD, call history + RCA, YAML editor.
- **`cli/`** — `agent setup|check|update|rca|dialplan|version`.

Config = 3-file deep merge: `config/ai-agent.yaml` (golden) ← `config/ai-agent.local.yaml` (overrides) + `.env` (secrets). Key source: `src/{engine,ari_client,rtp_server}.py`, `src/audio/`, `src/providers/`, `src/pipelines/`, `src/tools/`, `src/mcp/`.

## Asterisk integration — ARI-driven (the key difference vs AVR)

**Media transport is set up programmatically via ARI, not in the dialplan.** The dialplan only enters Stasis:
```asterisk
[from-ai-agent]
exten => s,1,NoOp(AI Voice Agent)
 same => n,Stasis(asterisk-ai-voice-agent)
 same => n,Hangup()
; per-agent override via channel vars before Stasis:
 same => n,Set(AI_PROVIDER=deepgram)
 same => n,Set(AI_AGENT=demo_deepgram)
 same => n,Set(AI_AUDIO_PROFILE=telephony_ulaw_8k)
```
`src/ari_client.py`: WS `…/ari/events?app=…&subscribeAll=true&subscribe=ChannelAudioFrame`; REST for `POST /channels`, `/answer`, `/bridges`, `/bridges/{id}/addChannel`, and **`POST /channels/externalMedia`**. On `StasisStart`: create bridge → create externalMedia channel → add both → answer.

```jsonc
POST /channels/externalMedia
{ "app":"<app>", "external_host":"host:port",
  "format":"ulaw", "direction":"both", "encapsulation":"rtp" }  // or audiosocket
```

**Two transport modes** (`config.audio_transport`):
| Mode | Protocol | Default port | Notes |
|---|---|---|---|
| **AudioSocket** | raw TCP | 8090 | best for remote engine/PBX split |
| **ExternalMedia RTP** | UDP/RTP | 18080 | pipeline-oriented; file-playback needs shared FS |

Required modules: `res_ari_applications.so`, `app_audiosocket.so`. ARI `:8088`.

> Caveat from research: the README says "AudioSocket via WebSocket" — that's inaccurate; AudioSocket is raw TCP, only ARI *events* use WS. AVA's AudioSocket type constants match upstream for audio (`0x10`) but it defines its own (`TYPE_UUID=0x01`, `TYPE_DTMF=0x03`, `TYPE_TERMINATE=0x00`, `TYPE_ERROR=0xFF`).

## Audio flow & formats

Inbound: Asterisk μ-law 8 kHz → AudioSocket/RTP → buffer (`1600` bytes = 200 ms) → VAD segmentation → resample if needed → STT/provider. Outbound: provider 16/24 kHz → resample → μ-law → framed back. Default μ-law@8k; optional slin16/PCM16@16k. **Resampling** (`src/audio/resampler.py`): NumPy linear interpolation replaced `audioop.ratecv` to kill ~0.3% drift / 50 Hz buzz; `audioop` still used for μ-law codec conversion. Profiles via `AI_AUDIO_PROFILE` (`telephony_ulaw_8k`, `wideband_pcm_16k`).

## Providers

Six production baselines: **OpenAI Realtime, Deepgram Voice Agent, Google Live (Gemini 2.0), ElevenLabs Conversational, Local Hybrid, Telnyx**. Two modes: **full-agent** (provider owns STT+LLM+TTS+VAD over its WS) vs **pipeline** (discrete STT/LLM/TTS). Local options: Faster-Whisper/Vosk/Sherpa (STT), llama.cpp/Ollama/Qwen (LLM), Piper/Kokoro/MeloTTS (TTS).

## VAD / barge-in (`VAD-BARGEIN-IMPROVEMENTS.md`)

Two-tier ownership: **full-agent providers own turn-taking** (platform is a safety layer); **hybrid/local pipelines** → platform is authoritative. Barge-in: provider event preferred (OpenAI `input_audio_buffer.speech_started`, ElevenLabs `interruption`); local WebRTC-VAD fallback only when media RX confirmed + agent speaking + caller-isolated audio. `_apply_barge_in_action` does **flush-only** (stop playback; does not kill provider session for Google/Deepgram). Knobs: `barge_in.{enabled,min_ms,energy_threshold,cooldown_ms}`.

## Tools / functions

Built-in telephony tools (all providers): `transfer`, `cancel_transfer`, `hangup_call`, `leave_voicemail`, optional `send_email_summary`. HTTP hooks: pre-call lookup, in-call HTTP tool, post-call webhook. MCP support + calendar tools. Pipeline mode needs an LLM with structured output.

## Limitations / notable decisions

- **ARI-orchestrated** transport (single Stasis entry; transport by config) — clean separation, easy multi-provider routing. **← the pattern most aligned with `node-asterisk-ari`.**
- **x86_64 / Linux / systemd only**; no ARM64/macOS/Windows.
- xAI Grok capped at 30-min sessions. Local LLM throughput modest.
- ExternalMedia-RTP + pipeline needs shared FS (Announcer-channel file playback) → AudioSocket preferred for remote splits. Their `docs/Transport-Mode-Compatibility.md` (Jan 2026) formalizes the validated transport×mode×playback matrix — mined for our own transport decision in [[12-ava-transport-compatibility-learnings]].

## What we borrow

- The **`StasisStart` → bridge → externalMedia channel → addChannel** recipe maps directly onto our [[02-asterisk-audio-integration-options#Option B — ARI ExternalMedia|Option B]] / [[02-asterisk-audio-integration-options#Option D — ExternalMedia/AudioSocket → publish into LiveKit room (gateway)|Option D]] gateway.
- **Resampling lesson**: use proper interpolation (or let LiveKit/Gemini handle 8k↔48k) — naive `ratecv` drift caused audible buzz.
- **Channel-variable routing** (`AI_PROVIDER`/`AI_AGENT`/`AI_AUDIO_PROFILE`) → our node payload sets the equivalent (`assistant_id`, `conversation_id`) before/at Stasis.
- **Barge-in via provider events, not local VAD** — matches Gemini Live's built-in turn detection; don't reinvent VAD.
