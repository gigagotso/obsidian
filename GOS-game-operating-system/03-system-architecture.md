---
title: System Architecture
type: architecture
tags: [gos, architecture, kernel, os]
created: 2026-06-13
---

# System Architecture

The "Operating System" metaphor, made literal. Borrowing OS concepts gives us a clean mental model **and** the exact isolation properties a gambling regulator demands.

## The big picture

```
┌──────────────────────────────────────────────────────────────┐
│                         PLAYERS / CLIENTS                       │
│            (web, mobile, live-table UI, voice)                  │
└───────────────────────────┬──────────────────────────────────┘
                            │  (rendered experience, streamed)
┌───────────────────────────┴──────────────────────────────────┐
│                        AI USERLAND  (untrusted)                 │
│  ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌───────────┐ │
│  │ Game        │ │ Dealer /   │ │ Narrative /  │ │ Difficulty│ │
│  │ Director    │ │ Opponent   │ │ Theme Engine │ │ / Pacing  │ │
│  │ agent       │ │ agent      │ │              │ │ agent     │ │
│  └────────────┘ └────────────┘ └──────────────┘ └───────────┘ │
│         emits INTENTS only ─ no money, no RNG, no DB           │
└───────────────────────────┬──────────────────────────────────┘
                            │  SYSCALL API (narrow, validated)
═══════════════════════════╪══════════════════════════════════  ← TRUST BOUNDARY
                            │
┌───────────────────────────┴──────────────────────────────────┐
│                      TRUST KERNEL  (certified)                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │ RNG /    │ │ Math /   │ │ Ledger / │ │ RG Guardian      │  │
│  │ Provably │ │ Game     │ │ Wallet / │ │ (limits, veto)   │  │
│  │ Fair     │ │ Manifest │ │ Settle   │ │                  │  │
│  │ service  │ │ validator│ │          │ │                  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Event Log / Deterministic Replay / Audit (append-only)    │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────┐
│        PLATFORM SERVICES (KYC/AML, payments, identity)         │
└──────────────────────────────────────────────────────────────┘
```

## The OS analogy, term by term

| OS concept | GOS equivalent | Why it matters |
|---|---|---|
| **Kernel** | Trust core: RNG, ledger, settlement, math validator, RG guardian | Deterministic, audited, never touched by AI |
| **Userland / processes** | AI agents (director, dealer, narrative, pacing) | Creative, untrusted, sandboxed |
| **Syscall** | The only API the AI may call to affect state (`requestDraw`, `placeBet`, `settle`, `escalate`) | Every privileged action is mediated & validated |
| **Driver** | Game format adapter (dice, cards, wheel, ladder) | Pluggable game primitives |
| **Process / session** | One live game instance for one player | Isolated, replayable |
| **Scheduler** | Turn/session orchestrator | Manages pacing, latency budget |
| **Memory protection** | Math-manifest validation of every AI intent | AI can't write outside its sandbox |
| **Filesystem** | Player model store + content library | Persistent state |
| **`init`** | Session bootstrap (seed commit, manifest load) | Establishes provable-fair seed up front |

## The two layers in detail

### AI Userland (untrusted, creative)
- A set of cooperating agents (see [[04-ai-game-engine]]).
- Has access to: player model (read-mostly), content primitives, conversation context.
- Has **no** access to: RNG internals, the ledger, raw odds, other players' data.
- Output: a stream of **intents** + presentation (text, persona behavior, theme choices).
- Treat exactly like untrusted user code. Assume it can be jailbroken/prompt-injected and design so that even a *fully compromised* userland cannot move money or change odds.

### Trust Kernel (certified, deterministic)
- **RNG / Provably-Fair service** — the only source of randomness. Commit-reveal seeds. ([[07-fairness-and-rng]])
- **Math / Game-Manifest validator** — each game ships a signed manifest declaring legal states, transitions, payout tables, max escalation, RTP. Every AI intent is checked against it. Out-of-bounds → rejected.
- **Ledger / Wallet / Settlement** — double-entry, append-only, the source of financial truth.
- **RG Guardian** — evaluates limits *before* any bet is accepted; can veto or end a session regardless of what the AI wants. ([[08-responsible-gambling]])
- **Event Log / Replay** — append-only log of every seed, intent, validation result, and settlement. Enables deterministic replay and audit. ([[09-regulatory-compliance]])

## The syscall contract (heart of the system)

The AI can only do four privileged things, each validated:

```
requestDraw(gameId, drawSpec)
   → kernel checks drawSpec ∈ manifest.allowedDraws
   → returns a provably-fair outcome from the COMMITTED seed
   → AI cannot retry / reroll (anti-"reroll until house wins")

placeBet(playerId, amount, market)
   → RG Guardian veto check (limits, affordability, self-exclusion)
   → ledger debit
   → returns accepted/rejected

escalate(sessionId, tier)
   → checks tier ≤ manifest.maxEscalation AND ≤ player's stake limits
   → adjusts stakes structure (NOT odds)

settle(sessionId, outcome)
   → kernel recomputes payout from manifest + drawn outcome
   → AI's claimed outcome is IGNORED; kernel is source of truth
   → ledger credit, event log append
```

Key property: **the AI proposes, the kernel disposes.** Even a malicious/hallucinating AI cannot pay itself or the player anything the certified math doesn't allow.

## Determinism & replay

Every session is reconstructable from `(committed_seed, ordered_event_log)`. Replaying yields byte-identical financial outcomes. The AI's creative output is logged but is *not* part of the financial replay — i.e. a dispute is resolved purely on kernel events, never on "what the AI said." This cleanly answers the regulator's first question: *"prove this game paid correctly."*

## Failure & fallback

- **Model down/slow** → scheduler falls back to a **scripted dealer** profile per game manifest; play continues, money path unaffected (Pillar 8, [[02-core-pillars]]).
- **AI emits garbage/injection** → manifest validator rejects; session pauses or reverts to scripted mode; incident logged.
- **Kernel is the only thing that must be highly available for correctness;** the AI layer is "nice to have" and degrades gracefully.

→ Next: [[04-ai-game-engine]]
