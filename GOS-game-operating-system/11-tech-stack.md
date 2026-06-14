---
title: Tech Stack
type: technical
tags: [gos, tech, implementation]
created: 2026-06-13
---

# Tech Stack

A proposed, opinionated implementation. The point isn't these exact tools — it's that the **trust boundary** from [[03-system-architecture]] maps onto real infrastructure with the right isolation and latency properties.

## Guiding constraints

1. **Hard isolation** between AI userland and trust kernel (separate services, separate trust zones, narrow API).
2. **Low latency** for live beats (Pillar 8) — streaming + pre-gen + fallback.
3. **Determinism & immutability** in the kernel (replay, audit).
4. **Horizontal scale** on the AI layer; **correctness-first** on the kernel.

## Layer-by-layer

### Trust Kernel — *correctness over everything*
- **Language:** something with strong correctness/auditability — **Rust** or Go. (Rust fits the deterministic, security-critical RNG/ledger nicely.)
- **RNG / provable-fair service:** CSPRNG + commit-reveal; hardware entropy; certified ([[07-fairness-and-rng]]).
- **Ledger:** double-entry, append-only. Event-sourced. Postgres (with strict constraints) or a purpose-built event store; consider an immutable/audit DB for the log.
- **Manifest validator:** loads signed manifests, validates every syscall intent.
- **RG Guardian:** in-kernel service, pre-bet veto path must be fast and always-on.
- **Determinism:** no wall-clock or ambient randomness in settlement logic — outcomes derive only from committed seeds + logged events.

### Syscall API — *the narrow waist*
- Strongly-typed, schema-validated RPC (gRPC / typed contracts).
- The **only** path from userland to privileged state.
- Every call: authenticated, validated against manifest, logged.

### AI Userland — *creativity, scale, sandbox*
- **Orchestration:** an agent framework coordinating Director / Dealer / Narrative / Pacing agents ([[04-ai-game-engine]]). Could be built on durable, stateful agent infra (e.g. Cloudflare Agents/Durable Objects, or a custom orchestrator) so each session is a stateful, isolated actor.
- **Models:** tiered — small/fast for frequent beats, stronger for arc decisions, tiny classifier for tilt/RG signals.
- **Structured output:** all intents are JSON-schema-validated; prose only on the player channel.
- **Caching/pre-gen:** vector store + cache of common beats; CDN-style delivery of pre-generated content.
- **Sandboxing:** userland runs with zero credentials to kernel internals; treat as hostile (prompt-injection assumption).

### Session runtime — *the scheduler*
- One **stateful session actor** per live game (Durable Object / actor model fits perfectly — isolation, single-threaded consistency, WebSocket-native).
- Holds session state, streams beats to the client, enforces the latency budget, manages scripted fallback.

### Client
- Real-time (WebSocket) web/mobile client.
- Renders streamed narrative + persona; exposes the **verify-fairness** UI (commit hash up front, seed reveal + recompute tool at session end).

### Platform services
- KYC/AML providers, payment rails, identity, geofencing ([[09-regulatory-compliance]]).
- Per-jurisdiction config service.

## Why an actor/Durable-Object model fits unusually well

A gambling session is exactly the thing actor systems are built for:
- **Isolation:** one player's session can't corrupt another's.
- **Statefulness:** persona memory, arc state, pacing live in the actor.
- **Consistency:** single-threaded per session avoids race conditions on bets/settlement.
- **Real-time:** WebSocket-native for live tables.
- **Hibernation:** idle sessions cost ~nothing; wake on the next beat.

(This is squarely in scope for the Cloudflare Agents / Durable Objects skills available in this environment, if prototyping there.)

## Observability & ops

- Full **event-log tracing** (kernel) — the audit trail doubles as observability.
- **AI quality monitoring:** are beats coherent? injection attempts? fallback rate?
- **RG dashboards:** intervention rates, tilt-signal accuracy.
- **Cost telemetry:** inference cost per session / per $ wagered ([[10-economy-and-monetization]]).

## Build vs. buy

- **Build:** the trust kernel (it's the moat + the cert surface), the manifest/intent system, the AI orchestration.
- **Buy:** KYC/AML, payments, base model inference, RNG *certification* (lab), infra.

→ Next: [[12-roadmap]]
