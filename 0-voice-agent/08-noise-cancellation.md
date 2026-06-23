# 08 — Noise Cancellation in the Browser / WebRTC Stack

What noise/echo handling we have today and what to implement, separately for the **sip.js web phone** (Web Call SIP) and the **pure-WebRTC LiveKit web call**. Closely tied to VAD/turn-taking ([[07-vad-and-turn-taking]]) — noisy audio wrecks endpointing and triggers false barge-ins. See [[00-index]], [[01-current-architecture]].

---

## The mental model: where audio can be cleaned

```
mic ─► [capture] ─► [browser APM: AEC + NS + AGC] ─► [optional custom DSP] ─► encode ─► network
                         ▲ getUserMedia constraints      ▲ Web Audio / TrackProcessor
                                                                                   │
   far end (human / Asterisk / LiveKit SFU) ◄── ... ◄── decode ◄────────────────────┘
                                                          │
                                       server-side denoise (RNNoise in voice-agent) — AI's STT only
```

Three families of processing, in order of where they run:

1. **Browser-native WebRTC APM** (Audio Processing Module) — `echoCancellation` (AEC3), `noiseSuppression` (NS), `autoGainControl` (AGC2). Free, built into Chromium/WebKit, toggled by `getUserMedia` constraints. Tuned for two-party speech; removes steady-state + much non-stationary noise; **AEC needs the far-end signal as reference**.
2. **Client-side ML denoise** — Krisp / RNNoise / deep models, inserted as an extra DSP node *before* the audio hits the peer connection. Catches what the native NS misses (keyboard, dog, café babble) and isolates the primary speaker.
3. **Server-side denoise** — RNNoise in the `docker-rtc` voice-agent (`audio_filter.py`). Only improves **what the AI hears (its STT)** — it does *not* clean what a human hears or what gets recorded, because it runs after fan-out.

> **Rule of thumb:** clean **client-side** as the primary defense (protects every downstream consumer — human, STT, recording, and saves uplink bandwidth). Use server-side only as a supplement for the AI's perception.

---

## Critical caveat: noise suppression vs STT/VAD quality

More denoise is **not** always better. Aggressive NS can:
- clip quiet speech / word onsets → STT misses words, VAD misses onsets (mid-sentence cutoffs, see [[07-vad-and-turn-taking]]);
- introduce musical-noise artifacts that some STT models dislike.

So: enable solid NS for **human ears**, but for the **AI's STT path** prefer a *speech-preserving* denoiser (RNNoise/Krisp are tuned for this) and validate transcription quality before/after. Don't stack three aggressive denoisers blindly (browser NS + client ML + server RNNoise can over-process).

Also: **AEC matters most for the AI agent.** If the agent's own TTS leaks back into the mic, it hears itself and interrupts itself. Browser AEC handles this when playback and capture share the WebRTC/Web Audio graph — but custom DSP or a separate `<audio>` playback path can *break the echo reference*. Keep playback inside the same audio context the AEC knows about.

---

## Path 1 — sip.js Web Phone (Web Call SIP, `server_id=1`)

Media path: **browser → Asterisk → far end** (PSTN / another agent). There is **no LiveKit track-processor pipeline here** — your options are browser-native APM and, if needed, a hand-wired Web Audio DSP node.

### What we have today
- `app/src/modules/pbx/webphone/services/call-management/constants.js` — `AUDIO_CONSTRAINTS = { echoCancellation:true, noiseSuppression:true, autoGainControl:true }`, passed via `sessionDescriptionHandlerOptions.constraints`.
- `app/src/config/sip.js` & `widget/src/config/sip.js` — `sessionDescriptionHandlerFactoryOptions.constraints`.
- `SipRegistrationService.js` / `MediaService.js` — `getUserMedia` with the three constraints.
- **Gap:** `app/src/modules/conversation-windows/composables/useMicrophone.js` calls `getUserMedia({ audio: true })` with **no constraints** → no NS/AEC/AGC on that path. Fix it to use the shared constraints.

### Level 1 — native APM (do this everywhere, it's free)
Ensure every `getUserMedia`/SDH config requests:
```js
const AUDIO_CONSTRAINTS = {
  echoCancellation: true,
  noiseSuppression: true,
  autoGainControl: true,
  // optional, newer Chromium — platform/ML voice isolation:
  // @ts-ignore
  voiceIsolation: true,
};
```
sip.js wiring (already the pattern in `constants.js`):
```js
userAgentOptions.sessionDescriptionHandlerFactoryOptions = {
  constraints: { audio: AUDIO_CONSTRAINTS, video: false },
};
// per-call: inviter.invite({ sessionDescriptionHandlerOptions: { constraints: { audio: AUDIO_CONSTRAINTS }}})
```
Audit: one shared `AUDIO_CONSTRAINTS` constant, imported by **every** capture site (close the `useMicrophone.js` gap).

### Level 2 — ML denoise for sip.js (only if native NS is insufficient)
sip.js has no processor API, so you insert a DSP node **manually** into the media graph before handing the track to the peer connection. Two ways:
1. **Custom `SessionDescriptionHandler`** (sip.js supports overriding the SDH factory) — capture the stream, run it through a Web Audio graph, and add the *processed* track to the `RTCPeerConnection`.
2. **MediaStream rewrite** — `getUserMedia` → `AudioContext` → `AudioWorkletNode` (WASM **RNNoise** or Krisp Web SDK) → `MediaStreamAudioDestinationNode` → use `.stream.getAudioTracks()[0]` as the sender track.

Sketch:
```js
const raw = await navigator.mediaDevices.getUserMedia({ audio: AUDIO_CONSTRAINTS });
const ctx = new AudioContext({ sampleRate: 48000 });
await ctx.audioWorklet.addModule('/rnnoise-worklet.js');     // WASM RNNoise (or @livekit/krisp)
const src = ctx.createMediaStreamSource(raw);
const denoise = new AudioWorkletNode(ctx, 'rnnoise');
const dest = ctx.createMediaStreamDestination();
src.connect(denoise).connect(dest);
const cleanTrack = dest.stream.getAudioTracks()[0];          // → feed to SDH / addTrack
```
Watch-outs: keep **`echoCancellation:true`** on the *raw* capture (your worklet won't do AEC); ensure remote audio plays through a path the browser AEC still references; mind the 48 kHz→8 kHz downsample Asterisk does (RNNoise runs at 48 kHz mono fine).

> For most SIP desk-phone use, **Level 1 is enough**. Reserve Level 2 for noisy contact-center environments.

---

## Path 2 — Pure WebRTC LiveKit Web Call (RTC widget + AI voice agent)

Media path: **browser → LiveKit SFU → (voice-agent / other participants)**. LiveKit has a **first-class track-processor pipeline**, so this path has the richest options.

### What we have today
- `widget/src/modules/ai-voice-agent/session/LiveKitSession.js` — `DEFAULT_AUDIO_OPTS = { echoCancellation:true, noiseSuppression:true, autoGainControl:true }`, applied via `room.localParticipant.setMicrophoneEnabled(true, DEFAULT_AUDIO_OPTS)`.
- Server-side **RNNoise** in `docker-rtc/voice-agent/audio_filter.py` on inbound audio to the agent.
- Robot Android uses native WebRTC AEC/NS/AGC **plus a custom noise-gate (`noiseGateDb`)**.

### Level 1 — native APM (already on)
Confirm `DEFAULT_AUDIO_OPTS` is passed on **every** publish (initial + device switches + unmute). Good today; keep it.

### Level 2 — LiveKit Krisp noise filter (recommended upgrade, drop-in)
LiveKit ships a client processor:
```bash
npm i @livekit/krisp-noise-filter
```
```js
import { KrispNoiseFilter, isKrispNoiseFilterSupported } from '@livekit/krisp-noise-filter';

const micTrack = await createLocalAudioTrack(DEFAULT_AUDIO_OPTS);
if (isKrispNoiseFilterSupported()) {
  await micTrack.setProcessor(KrispNoiseFilter());   // ML denoise + background-voice cancellation
}
await room.localParticipant.publishTrack(micTrack);
// toggle: const krisp = KrispNoiseFilter(); micTrack.setProcessor(krisp); krisp.setEnabled(false)
```
Krisp does **background-voice cancellation (BVC)** — removes *other people talking*, which native NS can't. Big win for open offices / call centers / robot-in-a-mall. (Licensing/bundle size: confirm terms; it's the LiveKit-blessed option.)

### Level 3 — custom track processor (parity with sip.js or special needs)
Implement LiveKit's `TrackProcessor` interface around a WASM RNNoise/AudioWorklet if you want a license-free denoiser or custom DSP (e.g. the robot's noise-gate). Same processor abstraction as Krisp.

### Level 4 — server-side (already present, AI-only)
RNNoise in the agent (`audio_filter.py`) cleans the **AI's STT input** only. Keep it as a backstop, but it's not a substitute for client-side NC for human-audible quality or recordings. If you adopt Krisp client-side, re-evaluate whether server RNNoise is still needed (avoid double-processing the AI path).

### LiveKit-Cloud option
LiveKit Cloud offers **enhanced/background noise & voice cancellation** as a managed feature (server-side, applied before fan-out). If we ever move to LiveKit Cloud it can replace some client work — but client-side still wins on bandwidth + protecting all consumers.

---

## Side-by-side

| | **sip.js Web Phone** | **LiveKit Web Call** |
|---|---|---|
| Media path | browser → Asterisk → far end | browser → LiveKit SFU → agent/peers |
| Native APM (AEC/NS/AGC) | ✅ via SDH constraints | ✅ via `setMicrophoneEnabled` opts |
| Drop-in ML denoise | ❌ none — hand-wire Web Audio | ✅ `@livekit/krisp-noise-filter` |
| Background-voice cancellation | only via custom WASM | ✅ Krisp BVC / LiveKit Cloud |
| Custom DSP (noise gate, RNNoise) | custom SDH / AudioWorklet | `TrackProcessor` interface |
| Server-side denoise | ❌ (Asterisk not wired for it) | ✅ RNNoise in voice-agent (AI only) |
| Effort for "good enough" | Level 1 (constraints) | Level 1 (already on) |
| Effort for "premium" | Level 2 custom worklet | Level 2 Krisp (one line) |

---

## Recommendations

1. **Unify the capture constraints.** One shared `AUDIO_CONSTRAINTS` with `echoCancellation/noiseSuppression/autoGainControl: true` (+ optional `voiceIsolation`), imported by **every** `getUserMedia` site. Close the `useMicrophone.js` gap. (Both paths.)
2. **LiveKit web call:** add **`@livekit/krisp-noise-filter`** as a track processor — biggest quality jump for least effort, and it does background-voice cancellation native NS can't. Make it a user toggle (some users prefer raw).
3. **sip.js web phone:** Level 1 is enough for most desk-phone use; add a **WASM RNNoise AudioWorklet** via a custom SDH only for noisy contact-center deployments needing parity.
4. **Protect the AI's STT:** prefer **speech-preserving** denoise (RNNoise/Krisp) over aggressive NS; keep **AEC on** so the agent doesn't hear its own TTS; A/B transcription quality before/after — over-denoising hurts STT and VAD ([[07-vad-and-turn-taking]]).
5. **Don't over-stack.** If client-side Krisp/RNNoise is on, reconsider the server-side RNNoise on the AI path to avoid double-processing.
6. **Keep playback in the AEC-known graph.** Custom DSP or out-of-band `<audio>` playback can break the echo reference — verify echo behavior after any DSP change.
