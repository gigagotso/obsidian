# 12 — AVA Transport-Mode Compatibility: learnings → our transport decision

Source: [AVA `docs/Transport-Mode-Compatibility.md`](https://github.com/hkjarral/AVA-AI-Voice-Agent-for-Asterisk/blob/main/docs/Transport-Mode-Compatibility.md) (updated Jan 2026, covers AVA v4.0 → v5.1.4). Read against [[02-asterisk-audio-integration-options#Deep dive: AudioSocket vs ExternalMedia RTP]], [[04-reference-ava]], [[09-standalone-agent-no-livekit]], [[11-unified-agent-architecture]].

> **Why this note exists:** notes 02/04/06 settled the *big* choice (Direction 2 / shared-core, no LiveKit) but left the transport sub-choice as "ExternalMedia, because ARI-native" with AudioSocket as the remote-split fallback. AVA's compatibility doc is a year of field validation on exactly this axis — and it both **sharpens** and **partly reverses** that lean.

---

## What the AVA doc actually establishes

AVA distinguishes three axes we had been folding into one: **transport** (AudioSocket TCP `8090` vs ExternalMedia RTP UDP `18080`), **provider mode** (full-agent STS vs modular STT→LLM→TTS pipeline), and **playback method** (streaming back over the media channel vs *file* playback via a separate Asterisk Announcer channel, controlled by `downstream_mode: stream|file` since v5.1.4).

Their validated matrix:

| Transport | Provider mode | Playback | Status |
|---|---|---|---|
| ExternalMedia RTP | Pipeline | **File** (Announcer) | ✅ validated (longest in production) |
| **AudioSocket** | **Full agent** | **Streaming** | ✅ validated (Deepgram, OpenAI Realtime, Google Live — Nov 2025) |
| AudioSocket | Pipeline | File (Announcer) | ✅ validated v4.0+ (was broken pre-Oct 2025) |
| Either | Pipeline | Streaming-first w/ file fallback | ⚠️ supported v5.1.4+, not the GA-stable path |

Five findings matter to us:

1. **The validated full-agent path is AudioSocket + streaming.** There is **no ExternalMedia + full-agent row in the matrix at all**. AVA never validated streaming TTS back over the RTP leg; their RTP deployments always return TTS via file playback on an Announcer channel. Our architecture is the full-agent/STS cell (OpenAI Realtime or Gemini Live, [[10-realtime-providers]]) — so AVA's field evidence for *our* cell is AudioSocket, not ExternalMedia.
2. **File playback is a shared-filesystem trap.** ExternalMedia+pipeline writes TTS to `audio/ai-generated/` and has Asterisk play the file — which **fails with "File does not exist"** the moment the engine runs on a different host than Asterisk, unless you NFS-mount the directory. AVA's own remote-deployment guidance is therefore "**use AudioSocket**". This doesn't bite our STS design (no file playback in the agent), but it's a landmine if we ever add a pipeline/file mode.
3. **The feedback-loop saga is an architecture lesson.** AVA's pre-v4.0 breakage (only the greeting heard, then silence) came from playing TTS through a *separate* Announcer channel in the same mixing bridge: the media channel then *hears the agent's own TTS in the bridge mix*, so they had to bolt on "gating" (mute STT while playing) and fight bridge-routing bugs for months (commits `fbbe5b9`, `181b210`, `fbaaf2e`). **If TTS is streamed back over the same media channel instead, Asterisk's mixer excludes a channel's own contribution automatically — no gating machinery needed.** Full-agent + streaming never had this bug class.
4. **AudioSocket format caveat:** AVA validates AudioSocket **only with `slin`** (signed linear 8 kHz) — consistent with the AudioSocket wire spec ([[02-asterisk-audio-integration-options#Option A — AudioSocket]]). So AudioSocket forfeits ExternalMedia-RTP's `format: ulaw` passthrough. The cost is one **byte-wise slin↔μ-law codec conversion** (table lookup) for OpenAI — *not* resampling; Gemini needs resampling under either transport.
5. **NAT is solvable on both** via bind-vs-advertise split (`host: 0.0.0.0` + `advertise_host`) — AVA's Milestone 23. But one outbound **TCP** connection (AudioSocket) remains categorically easier across Docker networks / VPNs than UDP RTP with advertised addresses.

---

## The correction to our earlier lean

[[02-asterisk-audio-integration-options]] leaned ExternalMedia "because `node-asterisk-ari` is an ARI bridge engine and we want mid-call escalation." That reasoning **conflated two independent axes**:

- **Who orchestrates** — dialplan (AVR-style) vs **ARI** (`Stasis` → bridge → media channel → `addChannel`). This is what gives us native escalation (swap the media channel for a human PJSIP endpoint, exactly `QueueAction`) and bridge recording.
- **What frames the wire** — RTP/UDP vs AudioSocket/TCP.

ARI orchestration does **not** require RTP. Two ARI-native ways to get AudioSocket framing:

```jsonc
// (a) ExternalMedia channel with AudioSocket encapsulation — a real channel in the bridge,
//     TCP AudioSocket framing on the wire (Asterisk 18+; we're 18+)
POST /channels/externalMedia
{ "app": "incoming", "external_host": "voice-agent:8090",
  "format": "slin", "encapsulation": "audiosocket", "direction": "both" }

// (b) ARI-originate a chan_audiosocket channel and add it to the bridge
POST /channels  { "endpoint": "AudioSocket/voice-agent:8090/<uuid>", ... }
```

Either way the media leg **is a channel object in our mixing bridge** — escalation, recording, snoop all stay native — while the wire is the dead-simple TCP framing AVA validated for full-agent streaming. This is in fact exactly what AVA itself does: its AudioSocket channel lives in the ARI bridge.

What ExternalMedia RTP still uniquely offers: `format: ulaw` **passthrough** to OpenAI (zero conversion), real RTP timestamps/media clock, and drop-late-packet semantics (no TCP head-of-line blocking). HOL blocking is negligible on a LAN/same-datacenter link — which is where our agent will sit relative to Asterisk — and only becomes real over lossy WANs, where UDP's advertise-host dance is its own pain anyway.

---

## Decision: which way we go

**Primary — ARI-orchestrated AudioSocket framing, streaming both ways.**
`node-asterisk-ari`'s `AIAgentAction` creates the mixing bridge and a `POST /channels/externalMedia` with `encapsulation: audiosocket, format: slin` (fallback: ARI-originated `AudioSocket/` channel) pointed at the `telephony-voice-agent` / `AsteriskTransport` ([[11-unified-agent-architecture]]). The agent streams TTS back **over the same socket** — never via file playback or a second channel.

| Requirement | How this satisfies it |
|---|---|
| ARI control (escalation, recording, snoop) | media leg is a bridge channel — `QueueAction`-style swap works unchanged |
| Field-validated for full-agent STS | AVA's ✅ AudioSocket + full-agent + streaming cell (Deepgram/OpenAI/Google Live) |
| Remote/containerized engine | one TCP port, no shared FS, no UDP advertise-host fragility |
| No feedback loop / gating machinery | same-channel streaming ⇒ bridge mixer excludes own audio (finding 3) |
| Implementation cost | framed TCP socket; no RTP stack, no jitter buffer |
| OpenAI Realtime (2b) | slin 8k ↔ μ-law is a byte-wise lookup — still no *resampling* |
| Gemini Live (2a) | resample 8k↔16k/24k — identical burden under either transport |

**Fallback — ExternalMedia RTP (`format: ulaw`), streaming TTS back over the RTP leg** if production shows TCP jitter artifacts, or if μ-law passthrough latency/CPU ever matters. Note we'd be *ahead* of AVA's validation there (they never shipped RTP + full-agent streaming) — acceptable as a fallback, not as the default.

**Never (for the agent path): file-based playback via Announcer channels.** It couples the agent to Asterisk's filesystem (finding 2) and reintroduces the self-hearing/gating bug class (finding 3). Flow announcements before/after the agent segment stay in `node-asterisk-ari`'s existing Playback — but **if any prompt must play *while* the agent session is live, route it through the agent's TTS/audio path, not an ARI Playback into the bridge**, or the agent will hear it.

### Spike checklist additions (extends [[05-implementation-plan]] watch-items)

1. Verify `externalMedia` + `encapsulation: audiosocket` on **our** Asterisk build (18+ required; confirm module `res_audiosocket`/`chan_audiosocket` loaded) — if quirky, fall back to ARI-originating `AudioSocket/…`.
2. Confirm bidirectional streaming: agent-sent slin frames reach the caller with no Announcer channel involved, and the socket's inbound frames contain caller-only audio (bridge does not echo the channel's own contribution).
3. Measure slin↔μ-law conversion + socket RTT vs the RTP path on one test call — keep the RTP fallback honest with a number.
4. Confirm continuous 20 ms frame cadence over a multi-minute call (AVA's pre-v4.0 "single-frame" symptom is the regression to watch for).
5. If engine and Asterisk ever split hosts: set `advertise_host`-equivalent and re-test — TCP should make this a non-event.

---

## What this does *not* change

- The **direction** ([[06-comparison-and-decision]]): Direction 2 / shared-core, no LiveKit, brain over HTTP — unchanged; AVA's doc actually re-confirms the shape (their full-agent mode *is* our architecture).
- ~~The **provider lever** ([[10-realtime-providers]]): OpenAI-direct (2b) stays the thin-loop favorite.~~ **Superseded — provider decision (Jul 2026): Gemini-direct (2a).** OpenAI Realtime was evaluated and ruled out; the phone agent uses **Gemini Live, the same engine as docker-rtc**. Implications for this transport decision:
  - The `AsteriskTransport` loop **owns resampling**: prefer `format: slin16` on the media channel so *Asterisk* delivers 16 kHz (Gemini's native input rate) — but AVA only validated AudioSocket with `slin`@8k, so the spike must test whether `encapsulation: audiosocket` honors `slin16`; if not, up-sample 8k→16k ourselves. Downsample Gemini's 24 kHz output with a proper resampler (AVA lesson: naive `ratecv` drift ⇒ audible buzz — [[04-reference-ava]]).
  - **Turn-taking** starts on Gemini Live's built-in VAD; if Georgian turn quality disappoints, the contingency is porting the silero + multilingual detector into the loop ([[07-vad-and-turn-taking]]). This is now the plan's main quality risk gate.
  - **Upside**: `agent-config` needs **no provider-agnostic refactor** — it is already Gemini-shaped (`gemini_api_key`, `model`); one provider across web + phone; ~5–6× cheaper audio tokens than OpenAI.
  - **Assistant binding**: the assistant is enabled/assigned at the **trunk channel-line level**, and the call is routed to it via the `AIAgentAction` flow node (node payload can override).
- The **builder/node design** ([[05-implementation-plan]]): transport remains invisible to the flow builder.
- [[11-unified-agent-architecture]]'s `audio/` layout: keep **both** `audiosocket.py` (primary) and `rtp.py` (fallback) behind the same `AsteriskTransport` interface — the decision above just sets which one `AIAgentAction` requests by default.
