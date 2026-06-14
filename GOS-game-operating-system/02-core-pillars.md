---
title: Core Pillars
type: principles
tags: [gos, principles, architecture]
created: 2026-06-13
---

# Core Pillars

The non-negotiable design principles. Every feature in the vault is justified against these. If a feature violates a pillar, it doesn't ship.

## Pillar 1 — Separation of Experience and Money (the prime directive)

> **The AI shapes how it feels. The kernel decides what it pays.**

A hard, audited boundary divides the system ([[03-system-architecture]]):
- **AI userland** controls: narrative, persona, pacing, theming, difficulty *structure*, which game format, bonus *triggers within math-defined bounds*.
- **Trust kernel** controls: RNG, every probability, the ledger, settlement, RTP, house edge, RG limits.

The AI can *ask* the kernel for an outcome via a narrow syscall, but cannot influence the distribution. This is what makes the whole thing certifiable. Without it, no regulator licenses you. See [[07-fairness-and-rng]].

## Pillar 2 — Provably Fair by default

Every outcome must be **independently verifiable** by the player after the fact (commit-reveal / verifiable RNG). "Trust us" is not a feature. Provable fairness becomes a **marketing weapon**, not just compliance. → [[07-fairness-and-rng]]

## Pillar 3 — Player welfare is kernel-level, not policy-level

Responsible-gambling limits (loss limits, time limits, cool-offs, affordability) live **in the kernel** and **override the AI**. The engagement-optimizing AI literally cannot route around them — they're enforced below it, like a memory protection boundary. → [[08-responsible-gambling]]

This also defends against the scariest failure mode: an AI that learns to exploit vulnerable players. It *structurally can't*, because the layer that could stop a session sits beneath the layer that wants to continue it.

## Pillar 4 — Generated, not authored

Content is **procedurally assembled** from primitives (formats × themes × personas × pacing), not hand-built per game. This is the scalability thesis: one engine → effectively infinite games. → [[04-ai-game-engine]], [[05-game-modes-catalog]]

## Pillar 5 — Deterministic replay & full auditability

Every session must be **deterministically replayable** from its seed + event log: same inputs → same outcome. Required for disputes, regulator audits, and debugging an AI that did something weird. The AI's *creative* choices are logged as events too. → [[03-system-architecture]], [[09-regulatory-compliance]]

## Pillar 6 — The AI is sandboxed and bounded

The AI never executes privileged operations. It emits **intents** ("escalate stakes to tier 3", "deal the next card", "switch theme to 'heist'") that the kernel **validates against the game's certified math manifest** before acting. An out-of-bounds intent is rejected, not executed. → [[04-ai-game-engine]]

## Pillar 7 — Persona ≠ advantage

The AI opponent can have personality, memory, and apparent skill, but in any player-vs-AI format the AI's "skill" must be **cosmetic or within published rules** — never secret information or odds manipulation. If the AI "reads" you, that read is flavor text over a fair draw. Transparency about this is a trust pillar. → [[05-game-modes-catalog]]

## Pillar 8 — Latency is a feature constraint

Live AI orchestration must fit a tight latency budget (a table feels broken if the dealer "thinks" for 4 seconds every turn). Architecture must support **pre-generation, streaming, and graceful fallback** to scripted behavior if the model is slow/unavailable. → [[11-tech-stack]]

## How the pillars relate

```
Pillar 1 (separation) ─┬─ enables → Pillar 2 (provable fair)
                       ├─ enables → Pillar 3 (welfare kernel)
                       └─ enables → Pillar 5 (auditability)

Pillar 4 (generated) ── powered by → Pillar 6 (bounded AI)
                                      Pillar 7 (fair persona)
                                      Pillar 8 (latency budget)
```

Pillar 1 is the root. Everything trustworthy flows from the experience/money split.

→ Next: [[03-system-architecture]]
