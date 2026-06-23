# 01 — Current LiveCaller Architecture

What exists today, across the three relevant repos. This is the ground truth the AI-agent feature has to plug into. See [[00-index]].

---

## 1. PHP API — PBX control plane (`livecaller-data/www/api`)

Laravel app. It is the **config & state plane** for telephony; it does not touch media.

### How it talks to Asterisk
| Channel | Direction | Purpose | Code |
|---|---|---|---|
| **ARI REST** | API → Asterisk | device state, recordings | `app/Services/PBX/AsteriskAriService.php` |
| **AMI socket** (PAMI) | Asterisk → API | manager events | `app/AMI/AMIConnection.php` |
| **Webhooks** (HMAC) | `node-asterisk-ari` → API | call lifecycle events | `app/Http/Controllers/Webhooks/PBXWebhookController.php` |
| **Realtime DB** | both | PJSIP config in `ps_*` tables | `app/Models/PBX/*`, `app/Models/Asterisk/*` |

Config: `config/pbx.php` (ARI host/port/user/pass), `config/livecaller.php` (`ami.*`, `sip.host`), `config/webhook-client.php` (`PBX_SIGNATURE_SECRET`).

### PBX webhook events handled (`Actions/PBX/*`)
`create_conversation`, `close_conversation`, `answer_conversation`, `join_queue`, `hold_conversation`/`unhold_conversation`, `transfer_to_operator`, `change_device_state`.

### The flow builder (the hook for our feature)
Scaffolding already exists — a visual IVR/call-flow builder:
- Tables: `pbx_flows`, `pbx_flow_nodes`, `pbx_flow_node_types` (migrations `2025_06_04_*`).
- Models: `app/Models/PBX/{Flow,FlowNode,FlowNodeType}.php`.
- Service/Controller: `app/Services/PBX/FlowService.php`, `app/Http/Controllers/PBX/FlowController.php` (`/pbx/flows`, `/pbx/flows/node-types`).
- A node has: `flow_id`, `type` (→ `pbx_flow_node_types.id`), `custom_name`, `x_axis`/`y_axis` (canvas), `extra_attributes['payload']` (node config).
- `pbx_flow_node_types.class_name` is the discriminator (e.g. `CallEmployeesAction`) — **this string is how `node-asterisk-ari` picks the handler.**

> **Our new node will be a new row in `pbx_flow_node_types`** (e.g. `class_name = 'AIAgentAction'`) plus a handler in `node-asterisk-ari`.

### Agents, queues, endpoints
- `User` (human agent) → `endpoints()` (MorphMany `Endpoint` = PJSIP extensions), `queueMembers()`, `departments()`.
- `queues` (PBX type via `extra_attributes['type']='pbx'`), `queue_members` (agent ↔ queue, `interface = PJSIP/{endpoint}`), `queue_has_users` pivot.
- `ps_endpoints/ps_aors/ps_auths/ps_registrations` carry a **`server_id` discriminator**: `1` = Web Call SIP, `2` = PBX. (migration `2025_06_28_*`)

### RTC integration already in the PHP API (important — we reuse this)
The web/robot voice agent is fully wired in PHP already:
- `app/Services/LiveKitTokenService.php` — JWT tokens + Twirp egress calls.
- `app/Services/RTCSessionService.php` — webhook handling, session/member tracking, transcripts, recording.
- `app/Http/Controllers/Webhooks/RTCWebhookController.php` — `/webhooks/rtc`, `/webhooks/rtc/transcript`.
- Internal endpoints the Python agent calls:
  - `GET /v1/internal/rtc/agent-config/{conversationId}` → system instruction, voice, model, tools, RAG config.
  - `POST /v1/internal/rtc/retrieve` → RAG vector search.
  - `POST /v1/internal/rtc/execute-tool/{conversationId}` → tool/function execution.
- Models `app/Models/RTC/{RTCSession,RTCMember,RTCSessionEvent}.php`, rooms named **`rtc-{conversationId}`**, identities `{memberType}-{memberId}` (e.g. `ai_assistant-456`, `visitor-123`).

---

## 2. `node-asterisk-ari` — PBX media/control engine (Node.js)

The **call-control layer**. Connects to Asterisk ARI and runs the flow engine. This is where the new AI node handler lives.

- ARI connect: `src/handlers/stasis.handler.js` via `ari-client`, two Stasis apps (`incoming`, `outgoing`). Ping/pong health + reconnect.
- Events: `StasisStart`, `StasisEnd`, `ChannelDestroyed`, `DeviceStateChanged`.
- Bridges: `src/services/bridge.service.js` — **mixing bridges** (caller ↔ agent). Originate: `src/services/originate.service.js`. Transfer, recording (`recording.service.js` → bridge WAV), hold music (`holdMusic.service.js`).
- Webhooks back to PHP: `src/services/webhook.service.js` (HMAC-SHA256).
- Redis: active call/queue state. MySQL (Drizzle): reads `pbx_flows`, `queues`, `ps_endpoints`, etc.

### Flow engine (the extension point)
`src/services/flows/flowExecutor.service.js` holds a `Map` keyed by `class_name`:
```js
this.nodeActions.set('IncomingCallEvent',  new IncomingCallEvent(this.client));
this.nodeActions.set('PlayAudioFileAction',new PlayAudioFileAction(this.client));
this.nodeActions.set('VoiceMenuAction',    new VoiceMenuAction(this.client)); // IVR/DTMF
this.nodeActions.set('CallQueueAction',    new QueueAction(this.client));      // route to humans
this.nodeActions.set('DialTrunkAction',    new DialTrunkAction(this.client));
this.nodeActions.set('WorkingHoursCondition', ...);
this.nodeActions.set('TrunkCondition', ...);
this.nodeActions.set('DialPatternMatchCondition', ...);
```
`executeNode()` looks up `node.nodeType.className` in that map and calls the handler.
Node actions live in `src/services/flows/nodes/actions/` and extend `BaseNode`.

> **Adding the AI node = one new file `AIAgentAction.js` + one `this.nodeActions.set('AIAgentAction', …)` line.** Everything else (flow plumbing, conversation webhooks) already exists.

### Crucially: there is **no media-streaming / AI code here yet**
No AudioSocket, no ExternalMedia, no RTP, no TTS/STT. Today the engine only *bridges channels*. The AI node is the first one that needs to send audio **off the bridge** to something else.

---

## 3. `docker-rtc` — the AI voice agent that ALREADY works (LiveKit + Gemini)

Pure WebRTC stack used by the web widget and the Android reception robot. **This is the thing we want to reuse for phone calls.**

- **LiveKit server** (Go SFU): WS signaling `:7880`, media UDP `50000–60000`, TCP fallback `7881`, Redis-coordinated, sends webhooks to `/webhooks/rtc`.
- **voice-agent** (`docker-rtc/voice-agent/agent.py`, Python, `livekit-agents` SDK):
  - `AgentServer.rtc_session` joins room `rtc-{conversationId}`.
  - Fetches config from `GET /v1/internal/rtc/agent-config/{id}`.
  - `google.realtime.RealtimeModel` = **Gemini 2.5 Flash native audio** (STS — speech-to-speech, built-in STT+TTS).
  - Silero VAD + **multilingual turn detector** (135M transformer), false-interruption handling.
  - **RNNoise** denoise on inbound audio (`audio_filter.py`).
  - Dynamic **tools** from assistant config → call back to `/v1/internal/rtc/execute-tool`.
  - **RAG** via `/v1/internal/rtc/retrieve`.
  - Streams **transcripts** to `/webhooks/rtc/transcript` (→ Pusher + `ConversationMessage`).
- **egress** worker: room composite recording → S3 (OGG/MP4).
- Identities: `visitor-{id}`, `ai_assistant-{id}`. Token = JWT (HS256) from `LiveKitTokenService`.

### Why this matters
The agent is **transport-agnostic about who the human is** — it just serves whoever is in the room. If a *phone* caller becomes a LiveKit participant in `rtc-{conversationId}`, this agent serves them with **zero changes**. That is the crux of [[05-implementation-plan]].

---

## 4. The AI "brain" — assistant model & internal contract (verified)

Everything LiveCaller-specific about the agent is **data + HTTP**, not LiveKit. This is what makes the agent reusable over any transport (see [[09-standalone-agent-no-livekit]]).

### The `AiAssistant` model (`app/Models/AI/AiAssistant.php`, table `ai_assistants`)
- `settings` (JSON): `name` (localized), **`system_instruction`**, **`voice`** (e.g. `Puck`), `channels`, `capabilities`.
- `integration_id` → `AiIntegration` → resolves **LLM provider + model** (`integration.model.model_name`, e.g. `gemini-2.5-flash-native-audio-preview-12-2025`) and the API key.
- `vector_store_id` → `VectorStore` → **RAG** source.
- `actions()` (pivot `ai_assistant_action`, with `is_active` + per-pivot `settings`) → **tools/functions** (`AiAction`, `type='tool'`, each with an `execution` config: url/method/headers/body/response_path).

### How a conversation picks its assistant (the keystone)
`RTCAgentConfigController` resolves the assistant via **`$conversation->aiAssistants->first()`** — a `morphedByMany` over the `conversation_members` pivot. **Therefore a PBX-origin conversation must have the assistant attached**, or `agent-config` returns 404. Two writes are needed (both verified against the code):
- `conversation->aiAssistants()->attach($id, ['member_type_id' => ConversationMemberType::getIdByName('ai_assistant'), …])` — read by `agent-config`.
- `conversation->extra_attributes['ai_assistant_id'] = $id` — read by `RTCSessionService::resolveTranscriptMember()` to attribute agent transcript turns.

### The internal HTTP contract (consumed by `agent.py`; reusable by any transport)
| Endpoint | Shape |
|---|---|
| `GET /v1/internal/rtc/agent-config/{conversationId}` | → `{ assistant_id, account_id, conversation_id, system_instruction, voice, model, gemini_api_key, rag:{enabled,assistant_id}, actions:[{name,description,parameters}] }` |
| `POST /v1/internal/rtc/retrieve` | `{ assistant_id, query, limit }` → `{ chunks:[{title,content,score}] }` |
| `POST /v1/internal/rtc/execute-tool/{conversationId}` | `{ assistant_id, action_name, arguments }` → `{ result }` |
| `POST /webhooks/rtc/transcript` | `{ room, identity, text, is_final, language }` → `ConversationMessage` + Pusher |

Auth = `X-Internal-Key` header. Identities are `{type}-{id}`: `ai_assistant-{assistantId}`, `visitor-{conversationId}` (the resolver remaps LiveKit's `agent-` → `ai_assistant-`). The payload is **Gemini-shaped today** (note the literal `gemini_api_key`); make it provider-agnostic to support OpenAI for telephony ([[10-realtime-providers]]).

---

## What's missing (the gap this project fills)

1. A flow-builder node type `AIAgentAction` (PHP seed + UI).
2. A handler in `node-asterisk-ari` that, instead of bridging to a human queue, hands the channel to the AI.
3. A **media bridge** from the Asterisk PJSIP channel to the AI runtime — see [[02-asterisk-audio-integration-options]].
4. Conversation/assistant wiring (the two writes above) so the phone call gets a `conversationId` + an attached assistant the existing contract understands.
