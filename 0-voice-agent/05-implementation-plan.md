# 05 — Implementation Plan: the "AI Voice Agent" flow node

How to add an **AI Agent action** to the PBX flow builder and bind a conversational voice agent to a phone call. This page is the LiveKit-based plan (Direction 1); the no-LiveKit variant lives in [[09-standalone-agent-no-livekit]] and shares almost all of Parts 1, 2, and 5 below. See [[00-index]] for the framing, [[01-current-architecture]] for the systems, [[02-asterisk-audio-integration-options]] for the bridge mechanics, [[10-realtime-providers]] for the model choice.

---

## The shape of the thing

The guiding principle, stated once: **the LiveCaller "brain" is transport-agnostic HTTP; we are only choosing how the caller's audio reaches it.** Everything LiveCaller-specific — which assistant answers, what it knows (RAG), what it can do (tools), what it said (transcripts) — is already exposed as internal HTTP endpoints and consumed by the existing web agent. Binding a phone call to an AI agent is therefore three jobs: (1) declare a flow-builder node, (2) handle that node in the call engine, and (3) get the audio from Asterisk into a place the agent serves.

```
Inbound SIP/PBX call
   │  Asterisk → Stasis "incoming" → node-asterisk-ari flow engine
   ▼
flow reaches the AIAgentAction node   (payload: { assistant_id, fallback_queue, … })
   │
   ├─ 1. ensure a Conversation (conversationId) and ATTACH the assistant to it
   ├─ 2. ensure LiveKit room rtc-{conversationId} + dispatch the agent
   ├─ 3. bridge the Asterisk channel into that room
   ▼
the SAME tuned Gemini agent serves the caller — RAG, tools, transcripts, recording all reused
   │
   └─ agent tool "transfer to human" → flow falls back to CallQueueAction(fallback_queue)
```

---

## Part 1 — PHP API: declare the node and bind the assistant

### 1a. Seed the node type
Node types live in `pbx_flow_node_types` and are discriminated by **`class_name`** — the exact string the call engine keys on. (Note: these rows are not currently seeded anywhere; `FlowService` references e.g. `CallEmployeesAction` by string, so add a proper seeder while you're here.) Insert:
```
type        = 'action'
name        = 'AI Voice Agent'
class_name  = 'AIAgentAction'      // ← the discriminator node-asterisk-ari matches
extra_attributes = { icon, category: 'ai' }
```

### 1b. Node payload (`pbx_flow_nodes.extra_attributes['payload']`)
```jsonc
{
  "assistant_id": "9876543210",   // an ai_assistants.id (drives prompt/voice/model/tools/RAG)
  "max_duration_sec": 1800,
  "fallback_queue": "uuid-…",     // optional: hand to humans on decline/timeout/tool-transfer
  "language": "ka"
}
```
The `assistant_id` references the existing **`AiAssistant`** model (`app/Models/AI/AiAssistant.php`, table `ai_assistants`): its `settings` JSON holds `system_instruction` and `voice`; `integration_id` resolves the LLM provider + model; `vector_store_id` enables RAG; `actions()` (pivot `ai_assistant_action`) are the tools. **No new AI config model is needed** — the phone path reuses the same assistant the web/widget uses.

### 1c. The critical binding: attach the assistant to the conversation
The web agent resolves *which* assistant to run by calling `GET /v1/internal/rtc/agent-config/{conversationId}`, which does `$conversation->aiAssistants->first()` — a `morphedByMany` over the `conversation_members` pivot. **So a PBX-origin conversation must have its assistant attached, or `agent-config` returns 404 "No AI agent assigned."** When the AI node fires (or in `Actions/PBX/CreateConversation` for AI-bound calls), do **both**:
```php
// 1) pivot attach — this is what agent-config reads
$conversation->aiAssistants()->attach($assistantId, [
    'member_type_id' => ConversationMemberType::getIdByName('ai_assistant'),
    'nickname'       => $assistant->setting('name.en'),
]);

// 2) extra_attributes — this is what the transcript resolver reads
$conversation->extra_attributes->set('ai_assistant_id', $assistantId);
$conversation->save();
```
The second write matters because `RTCSessionService::resolveTranscriptMember()` reads `conversation.extra_attributes['ai_assistant_id']` to attribute the agent's transcript turns. Set both and the existing transcript pipeline "just works" for phone.

> **One change to verify:** `RTCAgentConfigController` returns a Gemini-shaped payload (`gemini_api_key`, `model` from `integration.model.model_name`). It already resolves the assistant generically from the conversation, so a PBX-origin conversation works **as long as 1c is done**. If you later add OpenAI for telephony ([[10-realtime-providers]]), make this payload provider-agnostic.

### 1d. Ensure room + agent dispatch
Add (or extend) one internal endpoint the call engine calls to provision the session:
```
POST /v1/internal/rtc/ensure-agent-session
  { conversation_id, assistant_id, mode: "phone" }
  → { room: "rtc-9876543210", ws_url, dispatch: {...}, sip?: {...} }
```
It creates/loads the `RTCSession` (`mode: ["audio"]`, initiator = the assistant or the caller) via `RTCSessionService`, mints what LiveKit needs (`LiveKitTokenService`), and arranges agent dispatch (see Part 3).

---

## Part 2 — node-asterisk-ari: the node handler

### 2a. Register it
`src/services/flows/flowExecutor.service.js` keeps a `Map` of `class_name → handler`:
```js
import AIAgentAction from './nodes/actions/AIAgentAction.js';
// …
this.nodeActions.set('AIAgentAction', new AIAgentAction(this.client));
```

### 2b. `src/services/flows/nodes/actions/AIAgentAction.js`
A thin orchestrator (mirror `QueueAction` → `QueueBridgeManager`), delegating to a new `AIBridgeManager`. Responsibilities:
1. Read payload (`assistant_id`, `fallback_queue`, `max_duration_sec`).
2. Ensure the channel↔conversation correlation exists (the `create_conversation` webhook already runs; here we know the `conversationId`).
3. Call `POST /v1/internal/rtc/ensure-agent-session` → room/SIP info.
4. **Bridge** the channel (Part 3).
5. Wire termination: on `StasisEnd`/`ChannelDestroyed`, tear down the bridge/SIP leg; the existing `close_conversation` webhook fires as today.
6. Optional escalation: on a "transfer to human" signal (an agent tool), fall through to `CallQueueAction` with `fallback_queue`.

---

## Part 3 — the media bridge (LiveKit)

This is where Direction 1 commits to LiveKit. Research turned up two important realities that shape *how* you bridge (full detail in [[02-asterisk-audio-integration-options]] and the LiveKit-SIP notes in [[06-comparison-and-decision]]):

> ⚠️ **LiveKit SIP inbound cannot template the room name from a SIP header.** A dispatch rule's room name is either a fixed string (`dispatchRuleDirect`), per-caller (`dispatchRuleIndividual`), or derived from the **called number** (`dispatchRuleCallee`). There is no `roomName: "rtc-{X-Conversation-Id}"`. So you cannot, on a plain inbound call, land in `rtc-{conversationId}` purely from a header.

> ⚠️ **chan_pjsip ↔ LiveKit SIP interop is reportedly fiddly** (community reports favor chan_sip). LiveCaller is PJSIP-based — validate this early.

Given those, two viable LiveKit bridges:

### Primary — LiveKit *originates* the leg (explicit room name)
The clean way to get `rtc-{conversationId}` is to let **LiveKit place the call**, because the outbound `CreateSIPParticipant` API takes an **explicit `room_name`**:
1. AI node calls `ensure-agent-session`; PHP calls LiveKit `CreateSIPParticipant({ room_name: "rtc-{conversationId}", sip_call_to: <asterisk endpoint/DID for this channel>, participant_attributes: { conversationId } })` against an **outbound trunk pointed at Asterisk**.
2. LiveKit dials back into Asterisk; node-asterisk-ari bridges that inbound SIP leg to the original caller's channel (native ARI bridge).
3. Agent is dispatched to `rtc-{conversationId}` via the token/dispatch API; the existing Python agent serves it **unchanged** (it derives `conversationId` from the room name).

### Alternative — inbound + reconcile in the agent
1. Asterisk `Dial()`s a PJSIP endpoint at LiveKit SIP, setting `X-Conversation-Id` header; the inbound trunk maps it via `headers_to_attributes` into the **SIP participant's attributes**.
2. A `dispatchRuleDirect`/`dispatchRuleCallee` lands the caller in *some* room and dispatches the agent (`roomConfig.agents[]`).
3. The agent reads `conversationId` from `participant.attributes` instead of the room name (a small `agent.py` change) and calls `agent-config` with it.

Codecs/DTMF: configure Asterisk to offer **G.711 μ-law**; LiveKit SIP transcodes μ-law 8 k ↔ Opus 48 k automatically, so **no resampling lands on us** (a real win for staying on Gemini — see [[10-realtime-providers]]). DTMF via RFC 2833. Transfer via REFER (`TransferSIPParticipant`) requires Asterisk to honor REFER.

---

## Part 4 — reuse that comes for free

Because the caller becomes a LiveKit room participant in `rtc-{conversationId}`, the entire brain is reused with no new AI code (exact shapes in [[01-current-architecture]] and verified against the code):
- **Assistant config** — `GET /v1/internal/rtc/agent-config/{conversationId}` → `{ assistant_id, system_instruction, voice, model, gemini_api_key, rag, actions[] }`.
- **RAG** — `POST /v1/internal/rtc/retrieve` → `{ chunks: [{title, content, score}] }`.
- **Tools** — `POST /v1/internal/rtc/execute-tool/{conversationId}` `{ assistant_id, action_name, arguments }` → `{ result }`.
- **Transcripts** — `POST /webhooks/rtc/transcript` `{ room, identity, text, is_final, language }` → `ConversationMessage` + Pusher. Identity is `ai_assistant-{assistantId}` / `visitor-{conversationId}` (the resolver maps `agent`→`ai_assistant`).
- **Recording** — LiveKit egress → S3 (or fall back to native node-asterisk-ari bridge recording).

The only genuinely new AI-side work: a telephony **transfer/hangup tool** (an `AiAction` of type `tool`, like AVR's `avr_transfer`) and tuning greeting + barge-in for narrowband.

---

## Part 5 — frontend (flow builder)

- Add an **AI Voice Agent** node to the palette (`class_name = 'AIAgentAction'`).
- Config panel: assistant picker (existing `AiAssistant` list), language, max duration, optional fallback queue.
- Reuse the existing flow canvas (`pbx_flow_nodes` x/y). No new builder infrastructure.

The transport choice (LiveKit here, or the standalone gateway of [[09-standalone-agent-no-livekit]]) is **invisible to the builder** — the node payload is identical.

---

## Phased rollout

1. **Spike (days):** one DID → LiveKit (originate path) → existing agent in `rtc-{conversationId}`. **Goal: measure phone latency and confirm chan_pjsip↔LiveKit-SIP interop** before any builder work.
2. **Node + handler:** seed `AIAgentAction`, build `AIAgentAction.js` + `AIBridgeManager`, the assistant-attach in 1c, and `ensure-agent-session`.
3. **Telephony tool + escalation:** add `transfer`/`hangup` tools; wire fallback to `CallQueueAction`.
4. **Builder UI + GA.**
5. **Hardening:** barge-in/greeting tuning for 8 kHz, SIP trunk + NAT hardening, recording/transcript parity, load test (LiveKit SIP concurrency).

## Watch-items (carried forward to [[06-comparison-and-decision]])
- **Latency** of the SIP↔LiveKit↔Gemini path vs a direct gateway ([[09-standalone-agent-no-livekit]]).
- **chan_pjsip ↔ LiveKit SIP** interop (PJSIP is our stack).
- **Room-name mapping** — prefer the *originate* path so `rtc-{conversationId}` is explicit.
- **Georgian turn-taking** quality (covered in [[07-vad-and-turn-taking]] / [[10-realtime-providers]]).
- **`agent-config` for PBX origin** — works once 1c attaches the assistant; keep the payload provider-agnostic if OpenAI enters the picture.
