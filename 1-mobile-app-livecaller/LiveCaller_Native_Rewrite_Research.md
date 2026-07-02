# LiveCaller — Native App Stack Selection (by Product, by Operating System)

> **Status:** Decision-support document
> **Date:** 2026-06-29
> **Goal:** Build **one unified native app** containing three independent products,
> running on **iOS, Android, Windows, Linux, and macOS**.

---

## 0. TL;DR

- **One unified native app** bundles **three independent products/modules**:
  1. **Chat / Inbox** — the existing agent live-chat product.
  2. **SIP Phone** — inbound + outbound calls via **Asterisk** (integrated softphone).
  3. **LiveKit A/V/Screenshare** — audio, video, and screen-share product.
- **These three are completely separate at runtime.** The SIP Phone talks only to
  Asterisk; the LiveKit product talks only to LiveKit. They share no server, no
  connection, and no SDK. Routing between them is app logic only.
- **Web is already built and out of scope** for this work: the SIP webphone uses SIP.js,
  and the LiveKit product is already implemented for web. Both stay as they are.
- **Native targets:** iOS, Android, Windows, Linux, macOS.
- **Recommended stack for all five OSes: Flutter** — one codebase, with one library per
  product:
  - SIP Phone → **`sip_ua`** (SIP over WebSocket → Asterisk, plain WebRTC/DTLS-SRTP media)
  - LiveKit product → **`livekit_client`** (audio/video/screenshare)
  - Native call UI → **`flutter_callkit_incoming`** (CallKit / ConnectionService)
- **Not PJSIP / Linphone SDK:** wrong fit for a WebRTC-over-WebSocket client, no Dart
  bindings (5-OS cross-compile burden), and **copyleft licensing** (PJSIP GPL-2.0,
  Linphone AGPL-3.0) hostile to a closed-source product. Full reasoning + maintainer/
  health stats for every package in **§7 and §8**.

---

## 1. Current App State

| Aspect | Reality |
| :--- | :--- |
| Framework | Expo SDK 54, React Native 0.81.5, React 19, TypeScript |
| State / Nav | Redux Toolkit, Expo Router |
| Real-time | Laravel Echo + Pusher (WebSocket) for chat |
| Auth | OAuth + 2FA + Cloudflare Turnstile |
| Today's features | Login/2FA, Conversations, Inbox/chat, Visitor profiles, Profile |
| Size | ~4,000 LOC, ~35 files — small |
| Calling | None yet (chat only) |

The chat app is small enough to rewrite. The rewrite is justified by **adding the SIP
Phone and LiveKit products natively across 5 OSes**, not by the chat features.

---

## 2. The Three Products (separate by design)

```
┌──────────────────────── ONE UNIFIED NATIVE APP ────────────────────────┐
│                                                                         │
│  ┌────────────────┐   ┌────────────────────┐   ┌────────────────────┐  │
│  │ PRODUCT 1      │   │ PRODUCT 2          │   │ PRODUCT 3          │  │
│  │ Chat / Inbox   │   │ SIP Phone          │   │ LiveKit A/V/Share  │  │
│  │                │   │ (inbound+outbound) │   │                    │  │
│  │ Laravel Echo / │   │ sip_ua (SIP/WSS)   │   │ livekit_client     │  │
│  │ Pusker WS      │   │ → plain WebRTC     │   │ → LiveKit SFU      │  │
│  └───────┬────────┘   └─────────┬──────────┘   └─────────┬──────────┘  │
│          │                      │                        │             │
└──────────┼──────────────────────┼────────────────────────┼─────────────┘
           ▼                      ▼                        ▼
      Pusher / API           Asterisk                 LiveKit SFU
                          (WSS + DTLS-SRTP)        (audio/video/share)
       — three independent backends; nothing shared between them —
```

| Product | Backend | Client library | Purpose |
| :--- | :--- | :--- | :--- |
| **1. Chat / Inbox** | Pusher / REST API | Echo + Pusher client | Live-chat agent inbox (exists today) |
| **2. SIP Phone** | **Asterisk** (SIP over WSS) | **`sip_ua`** + WebRTC media | Inbound + outbound phone calls |
| **3. LiveKit A/V** | **LiveKit SFU** | **`livekit_client`** | Audio, video, **screenshare** |

> **Important:** Product 2 (SIP) and Product 3 (LiveKit) are unrelated. The SIP call's
> media is **plain WebRTC (DTLS-SRTP) straight to Asterisk and never touches LiveKit.**
> They only coincidentally both sit on the framework's low-level WebRTC plugin
> (`flutter-webrtc`) — a shared dependency, not a shared connection.

**Web is already implemented for Products 2 and 3** (SIP.js webphone; LiveKit web) and is
not part of this native rewrite.

---

## 3. Building Blocks (what the app needs, per product)

| Product | Needs | Supplied by (Flutter) |
| :--- | :--- | :--- |
| Chat / Inbox | WebSocket realtime + REST | `laravel_echo`-style client / `pusher` / `dio` |
| SIP Phone | SIP signaling over WSS + WebRTC media + call UI/wake | `sip_ua` + `flutter-webrtc` + `flutter_callkit_incoming` |
| LiveKit A/V | Room join + audio/video + **screen capture** | `livekit_client` (+ platform screen-capture) |

### Role of each library — and exactly where it's needed

`flutter_callkit_incoming` is **not** a media library. It bridges to the OS telephony
frameworks (**CallKit** on iOS, **ConnectionService/Telecom** on Android) to provide the
**native incoming-call experience**: full-screen call UI over the lock screen, system
ringtone, Accept/Decline, audio-session priority/routing (Bluetooth/CarPlay), call
history, and Focus/DND handling. It is the piece that **wakes and presents a call when the
app is backgrounded or killed**.

| Library | Layer | Where it's needed | Where it's NOT needed |
| :--- | :--- | :--- | :--- |
| `sip_ua` | SIP signaling | All platforms (the SIP Phone) | — |
| `flutter-webrtc` | Media transport | All platforms (under SIP + LiveKit) | — |
| `livekit_client` | A/V/screenshare | All platforms (LiveKit product) | — |
| `flutter_callkit_incoming` | Native call UI / wake | **SIP Phone inbound on iOS + Android only** | **Desktop** (no CallKit/ConnectionService → use in-app UI + OS notifications); outbound-only flows; LiveKit sessions that are always joined from inside the app |

**Why it's mandatory on iOS:** on iOS 13+, when a PushKit VoIP push wakes the app for an
incoming call you **must report it to CallKit immediately**, or iOS **terminates the app
and eventually stops delivering your VoIP pushes**. There is no compliant way to show an
incoming SIP call from a killed app without CallKit. On Android it's strongly preferred
(native call UI, lock-screen, audio focus) but not kill-or-be-killed.

**It is isolated and swappable:** the call-UI layer can be replaced without touching
`sip_ua` or `livekit_client`. See §8 for alternatives.

---

## 4. Stack Selection — Per Operating System

The same evaluation runs on every OS; each table covers all three products.

### 4.1 iOS

| Stack | Chat | SIP Phone | LiveKit A/V/Share | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Flutter** ✅ | Echo/Pusher | `sip_ua` | `livekit_client` (+ ReplayKit broadcast for screenshare) | One codebase |
| React Native | works | SIP.js + `react-native-webrtc` | LiveKit RN SDK | Also works on iOS |
| Native (Swift) | works | **no native SIP-over-WS lib** → wrap/WebView | LiveKit Swift SDK | SIP leg is the weak point |

- **Pick: Flutter.** Call UI via **CallKit**; incoming calls woken by **PushKit VoIP
  push** (report to CallKit immediately or iOS kills the app). **Screenshare** needs a
  ReplayKit **Broadcast Upload Extension** — verify in spike.

### 4.2 Android

| Stack | Chat | SIP Phone | LiveKit A/V/Share | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Flutter** ✅ | Echo/Pusher | `sip_ua` | `livekit_client` (+ MediaProjection screenshare) | One codebase |
| React Native | works | SIP.js + `react-native-webrtc` | LiveKit RN SDK | Works |
| Native (Kotlin) | works | **no native SIP-over-WS lib** → wrap/WebView | LiveKit Android SDK | SIP leg weak |

- **Pick: Flutter.** Call UI via **ConnectionService/Telecom**; incoming calls woken by
  **FCM high-priority push → Foreground Service**
  (`foregroundServiceType="microphone|camera|phoneCall"`). **Screenshare** via
  **MediaProjection** + a media-projection foreground service.

### 4.3 macOS

| Stack | Chat | SIP Phone | LiveKit A/V/Share | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Flutter (desktop)** ✅ | Echo/Pusher | `sip_ua` | `livekit_client` (native window/screen capture) | Reuses codebase |
| Native (Swift) | works | no native SIP-over-WS → wrap | LiveKit Swift SDK | Separate codebase |
| React Native macOS | partial | SIP.js | LiveKit | rn-macos lags |

- **Pick: Flutter (desktop build).** No background-kill, so call lifecycle is simpler;
  a persistent WSS keeps the SIP phone registered. Screenshare is first-class on desktop.

### 4.4 Windows

| Stack | Chat | SIP Phone | LiveKit A/V/Share | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Flutter (desktop)** ✅ | Echo/Pusher | `sip_ua` | `livekit_client` (screen capture) | Reuses codebase |
| Native (C#/WinUI 3) | works | no native SIP-over-WS → wrap | LiveKit web/Rust/C++ | Separate codebase + language |
| React Native Windows | partial | SIP.js | LiveKit | rn-windows lags |

- **Pick: Flutter (desktop build).** Avoids a separate C#/WinUI codebase; same simple
  desktop lifecycle and first-class screenshare.

### 4.5 Linux

| Stack | Chat | SIP Phone | LiveKit A/V/Share | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Flutter (desktop)** ✅ | Echo/Pusher | `sip_ua` | `livekit_client` (screen capture) | Only single-codebase option |
| Native (C++/Qt/GTK) | works | raw SIP libs | LiveKit C++/Rust | Heavy, separate codebase |
| React Native | ❌ no Linux | — | — | Rules RN out as the sole stack |

- **Pick: Flutter (desktop build).** Linux is the deciding constraint — **React Native
  can't target it** and native means a full C++/Qt stack. Flutter covers it from the same
  codebase. Screenshare under Wayland/X11 — verify in spike.

---

## 5. Cross-OS Summary

| OS | Stack | Chat | SIP Phone | LiveKit A/V/Share | Wake / call UI |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **iOS** | Flutter | ✅ | `sip_ua` | `livekit_client` + ReplayKit | CallKit + PushKit |
| **Android** | Flutter | ✅ | `sip_ua` | `livekit_client` + MediaProjection | ConnectionService + FCM + FGS |
| **macOS** | Flutter desktop | ✅ | `sip_ua` | `livekit_client` + screen capture | tray/notifications |
| **Windows** | Flutter desktop | ✅ | `sip_ua` | `livekit_client` + screen capture | tray/notifications |
| **Linux** | Flutter desktop | ✅ | `sip_ua` | `livekit_client` + screen capture | tray/notifications |

**One stack — Flutter — wins on every OS.** Same three product libraries everywhere; the
only per-OS differences are the call-wake mechanism and the screenshare capture API.

### Why not the alternatives (one line each)
- **React Native:** strong on iOS/Android but **no Linux** and weak Win/macOS desktop →
  can't be the sole stack for a 5-OS unified app.
- **Kotlin Multiplatform / fully native:** reaches all OSes for UI, but the **SIP Phone**
  has **no first-class native SIP-over-WebSocket library** → you'd embed SIP.js in a
  WebView or wrap a Dart/JS lib, defeating the point of native and adding the most risk.

---

## 6. Recommended Architecture (Flutter, unified app)

```
┌──────────────────────────────────────────────────────────────┐
│  Shared app shell: auth (OAuth/2FA), navigation, state         │
├──────────────┬───────────────────────┬───────────────────────┤
│ Chat / Inbox │  SIP Phone module      │  LiveKit module        │
│ Echo/Pusher  │  sip_ua → Asterisk     │  livekit_client → SFU  │
│              │  (plain WebRTC)        │  (A/V/screenshare)     │
└──────────────┴───────────┬───────────┴───────────┬───────────┘
                           │                        │
        Call orchestration (push wake, CallKit/ConnectionService,
        foreground service, audio session) — shared by both call modules
```

**Incoming SIP call lifecycle (mobile):**
```
Inbound call at Asterisk → backend wake push
  iOS: PushKit VoIP push   |   Android: FCM high-priority push
    → app wakes → report to CallKit / ConnectionService IMMEDIATELY
    → user answers → sip_ua answers over WSS → plain WebRTC media (DTLS-SRTP)
```
> Mobile: **register the SIP phone on demand via push** — don't keep `sip_ua` registered
> 24/7 (battery + OS kills background sockets). Desktop: a persistent WSS registration is fine.

---

## 7. Why NOT PJSIP or Linphone SDK (for this architecture)

PJSIP and Linphone are excellent **native C/C++ SIP+media stacks** — but they are the
wrong tool for **a WebRTC SIP client that talks to Asterisk over WebSocket**, which is the
architecture LiveCaller already uses on web (SIP.js). Six concrete reasons:

### 1. Architecture mismatch (the big one)
Our SIP Phone is a **WebRTC endpoint**: signaling over **WSS**, media as **DTLS-SRTP**
negotiated the WebRTC way (ICE, rtcp-mux, bundle), terminating on Asterisk's
`chan_pjsip` **WebRTC** profile. PJSIP/Linphone are designed to **own the media
themselves** (their own RTP stack, codecs, audio devices). To use them as a WebRTC client
you must either run their media stack (which does **not** interop with browser-style
WebRTC cleanly) or **disable their media and hand the SDP to a separate WebRTC engine** —
the fragile "SDP interception" hack. Both are dead ends compared to a library built for
SIP-over-WS + WebRTC from the start (`sip_ua`, which is a Dart port of JsSIP).

### 2. No native SIP-over-WebSocket
**PJSIP has no SIP-over-WebSocket transport** (UDP/TCP/TLS only). Asterisk's WebRTC
endpoints require WSS signaling, so PJSIP can't even connect the way our web phone does
without custom transport patches. Linphone's WS support exists but is not its primary,
well-trodden path.

### 3. No Dart/Flutter bindings → a cross-compile burden on 5 OSes
Both are C/C++ with no Flutter bindings. You'd write FFI/platform-channel glue **and
cross-compile static libraries for every target** (iOS arm64, all Android NDK ABIs,
Windows, Linux, macOS), then maintain that build matrix forever. `sip_ua` is **pure Dart**
and the only native dependency is `flutter-webrtc` — which is already a maintained,
prebuilt plugin for all those platforms.

### 4. Licensing is hostile to a closed-source product
- **PJSIP = GPL-2.0** (dual-licensed; commercial license sold by Teluu).
- **Linphone SDK = AGPL-3.0** (dual-licensed; commercial license sold by Belledonne).

Both are **strong copyleft**. Statically linking them into a proprietary app and
distributing it (App Store / Play Store / desktop installers) triggers source-disclosure
obligations — **unless you buy a commercial license** (recurring cost + negotiation).
AGPL is the most aggressive (its network-use clause). GPL also conflicts with Apple App
Store terms in practice. By contrast `sip_ua`, `flutter-webrtc`, `livekit_client`, and
`flutter_callkit_incoming` are **MIT / Apache-2.0** — permissive, zero obligation.

### 5. Divergence from the working web phone
Web already runs **SIP.js over WSS**. `dart-sip-ua` is a **direct Dart port of JsSIP**, so
it produces the same SIP/SDP behavior against the **same Asterisk WebRTC config** — you
get parity with the proven web client. PJSIP/Linphone negotiate calls differently (SDP,
ICE, codecs), forcing you to maintain **two divergent SIP implementations** and two sets
of Asterisk edge cases.

### 6. Security surface & patching
PJSIP has a long history of media-parsing **CVEs**; owning the native binary means you own
patching it across 5 platforms. With the WebRTC-engine approach, the security-sensitive
media code is Google's **libwebrtc** (inside `flutter-webrtc`), which is funded and patched
upstream.

### When PJSIP/Linphone *would* be right
A native softphone over **UDP/TCP/TLS** (not WebSocket), deep telephony features (presence,
complex transfer/conferencing handled client-side), or a non-WebRTC media path. **None of
those apply here** — Asterisk is configured for WebRTC and the web phone already proves the
SIP-over-WS + WebRTC model.

---

## 8. Package Maintainer & Health Stats (as of 2026-06)

GitHub figures pulled 2026-06-29. "Health" is my read of maintenance risk for production.

### Recommended stack (Flutter)

| Package | Maintainer / Owner | License | ⭐ | Contributors | Last commit | Health |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`flutter-webrtc`** | flutter-webrtc org (lead: Duan Weiwei / *cloudwebrtc*) | **MIT** | ~4.5k | ~159 | 2026-06 (active) | 🟢 Strong foundation; both other libs depend on it |
| **`livekit_client`** | **LiveKit Inc** (company) | **Apache-2.0** | ~406 | ~42 | 2026-06 (daily) | 🟢 Company-backed, very active |
| **`sip_ua`** (dart-sip-ua) | flutter-webrtc org (*cloudwebrtc* et al.) | **MIT** | ~373 (338 forks) | ~52 | **2025-12 (~6 mo)** | 🟡 Community-run; JsSIP port; somewhat stale, 170 open issues |
| **`flutter_callkit_incoming`** | **hiennguyen92** (single user) | **MIT** | ~242 (519 forks) | ~52 | 2026-06 (active) | 🟡 Single-maintainer; 216 open issues; widely used |

### Alternatives considered

| Package | Maintainer / Owner | License | ⭐ | Contributors | Last commit | Note |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `pjsip/pjproject` | **Teluu Ltd** (Benny Prijono) | **GPL-2.0** + commercial | ~2.6k | ~149 | 2026-06 (active) | Healthy project, **wrong fit + copyleft** |
| `linphone-sdk` | **Belledonne Communications** | **AGPL-3.0** + commercial | ~147* | ~149 | 2026-06 (active) | *GitHub mirror; real dev on gitlab.linphone.org. **AGPL + heavy** |
| `SIP.js` | OnSIP | MIT | ~2.1k | — | 2026-06 | Web reference (already in use) |
| `JsSIP` | Versatica (Iñaki Baz Castillo) | MIT* | ~2.6k | — | 2026-05 | *custom permissive; `dart-sip-ua` is its Dart port |
| `react-native-webrtc` | react-native-webrtc org | MIT | ~5.0k | — | 2026-06 | For the RN path (not chosen — no Linux) |
| `livekit/client-sdk-react-native` | LiveKit Inc | Apache-2.0 | ~280 | — | 2026-06 | RN LiveKit (not chosen) |

### Call-UI layer: alternatives to `flutter_callkit_incoming`

This is the one 🟡 dependency on the critical path, so here are the real options
(stats 2026-06-29):

| Package | Owner | License | ⭐ | Contrib | Last commit | Assessment |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`flutter_callkit_incoming`** ✅ | hiennguyen92 (user) | MIT | ~242 (519 forks) | ~52 | **2026-06 (active)** | Most popular & most actively maintained; highest-level API (renders the incoming-call UI straight from a push payload — ringtone, avatar, custom buttons). Single *owner* but 52 contributors. **Best default.** |
| `callkeep` | flutter-webrtc org | MIT | ~155 (177 forks) | ~18 | **2025-03 (~15 mo stale)** | Natural pairing with `sip_ua` (same org) and a cleaner low-level API, **but currently under-maintained.** Good fallback *if* it gets attention; you build more of the UI orchestration yourself. |
| `connectycube_flutter_call_kit` | ConnectyCube (org) | Apache-2.0 | ~65 (94 forks) | ~7 | 2025-10 (~8 mo) | FCM-incoming-call focused; tied to ConnectyCube's ecosystem. Usable standalone but niche. |
| `webtrit_callkeep` | WebTrit (org) | **none declared** ⚠️ | ~9 | ~5 | 2026-06 (active) | Purpose-built by a **SIP softphone vendor** (closest domain match) and actively developed, but **tiny adoption and no license** (legal blocker until they add one). Watch, don't adopt yet. |
| **Roll your own** | you | n/a | — | — | — | Thin platform channels to **CallKit (Swift)** + **ConnectionService (Kotlin)** directly. **Zero third-party risk on the most lifecycle-critical piece**, full control — at the cost of writing/owning ~2 small native modules. Reasonable for a serious long-lived product. |

**Verdict:** **No clearly better off-the-shelf option than `flutter_callkit_incoming`
today** — it's the most popular, most actively maintained, and most feature-complete.
`callkeep` would be the elegant same-org pairing but is ~15 months stale, so it's a
fallback, not an upgrade. If you want to eliminate third-party risk on this critical
piece, **rolling a thin native CallKit/ConnectionService module is the genuinely "better"
long-term option** (more work, total control). Because the layer is swappable, you can
**ship on `flutter_callkit_incoming` and migrate later** if needed.

### Honest risks in the recommended stack (and mitigations)
- **`sip_ua` is community-maintained and ~6 months since last commit.** Mitigations: it's
  a thin JsSIP port (protocol is stable), **MIT-licensed so forkable**, the heavy/
  security-critical media work lives in the very-active `flutter-webrtc`, and the same org
  maintains both. **Action:** in the spike, confirm it handles your Asterisk dialog edge
  cases (re-INVITE/hold, transfer, DTMF); budget for a maintained fork if needed.
- **`flutter_callkit_incoming` is single-maintainer with many open issues.** Mitigations:
  MIT, large user base, active. **Action:** evaluate `callkeep`-style alternatives as a
  backup; the call-UI layer is replaceable without touching SIP/LiveKit logic.
- **`livekit_client` and `flutter-webrtc` are low-risk** (company- and org-backed,
  permissive, actively developed).

---

## 9. Roadmap (de-risk before committing)

1. **Per-product browser sanity check first** (no app work):
   - SIP: confirm Asterisk WSS softphone works (you already have the SIP.js webphone).
   - LiveKit: confirm the existing web A/V/screenshare product as the reference.
2. **Flutter spike (days), one screen per product:**
   - `sip_ua` registers to Asterisk and completes an inbound + an outbound call.
   - `livekit_client` joins a room with audio, video, **and screenshare**.
   - Run on **desktop targets** (Win/Linux/macOS), not only mobile.
3. **Mobile first (iOS + Android):** hardest part is the SIP incoming-call lifecycle
   (push wake + CallKit/ConnectionService + foreground service) and mobile screenshare
   (ReplayKit / MediaProjection).
4. **Port the existing chat app** into the unified Flutter app — small (~4k LOC).
5. **Desktop last:** simpler lifecycle; screenshare is first-class there.

---

## 10. Open Questions

1. Confirm in the spike: Flutter desktop builds of `sip_ua` + `livekit_client` on
   Win/Linux/macOS, and **screenshare** on each (ReplayKit, MediaProjection, Wayland/X11).
2. SIP wake transport: PushKit (iOS) + FCM (Android); reuse Pusher/Echo for in-app state.
3. Desktop priority: launch requirement or fast-follow after mobile?
4. LiveKit scope on mobile: 1:1 vs multi-party; is mobile **screenshare** required day one?
5. Do chat + SIP + LiveKit share one auth/session, or separate sign-ins per product?

---

## Sources
- [sip_ua — pub.dev](https://pub.dev/packages/sip_ua) ·
  [dart-sip-ua repo](https://github.com/flutter-webrtc/dart-sip-ua)
- [SIP.js](https://sipjs.com/) · [Asterisk WebRTC config](https://www.siperb.com/kb/article/asterisk-webrtc/)
- [LiveKit SDK platforms](https://docs.livekit.io/transport/sdk-platforms/) ·
  [Flutter SDK](https://github.com/livekit/client-sdk-flutter) ·
  [React Native SDK](https://github.com/livekit/client-sdk-react-native)
- [flutter_callkit_incoming](https://pub.dev/packages/flutter_callkit_incoming) ·
  [react-native-callkeep](https://github.com/react-native-webrtc/react-native-callkeep)
