---
title: Fairness & RNG
type: design
tags: [gos, fairness, rng, math, trust]
created: 2026-06-13
---

# Fairness & RNG

The kernel's beating heart ([[03-system-architecture]]). This is what makes "an AI is playing gambling with you" trustworthy instead of terrifying: the AI cannot touch any of this.

## The threat the design must kill

The instinctive fear of AI gambling: *"the AI will rig it against me / learn to exploit me."* The architecture must make that **impossible by construction**, not improbable by policy. Two mechanisms:

1. **Provable fairness** — the player can verify every outcome themselves.
2. **The experience/money split** ([[02-core-pillars]] Pillar 1) — the AI never has odds authority to begin with.

## Provably-fair scheme (commit–reveal)

Per session (or per game), before any play:

```
1. COMMIT
   server_seed       = secure random (kernel)
   commit            = hash(server_seed)          → shown to player UP FRONT
   client_seed       = player-provided or client random
   nonce             = incrementing counter per draw

2. DRAW (each outcome)
   outcome = deterministic_fn(server_seed, client_seed, nonce)
           mapped through the certified payout table
   nonce++

3. REVEAL (session end)
   server_seed revealed → player checks hash(server_seed) == commit
   player can recompute EVERY outcome and confirm nothing was altered
```

Properties this guarantees:
- The server **could not** have chosen outcomes after seeing bets (seed committed first).
- The player's `client_seed` means the server alone can't have engineered a sequence.
- **The AI cannot reroll**: a draw consumes a nonce; the kernel exposes no "try again" — every nonce is accounted for in the reveal (an extra/missing nonce is detectable).

This is industry-proven (crypto casinos) and becomes a **marketing asset**: *"Don't trust our AI? Verify the math yourself."*

## Certified math: the Game Manifest

Each game ships a signed **manifest** the kernel enforces:

```
manifest:
  gameId, version, signature
  rtp:                e.g. 0.96   (return-to-player)
  houseEdge:          1 - rtp
  payoutTable:        outcome → multiplier
  allowedDraws:       which draw specs are legal
  maxEscalation:      cap on stake tiers the AI may request
  volatility:         distribution shape (for certification)
  rgProfile:          required RG hooks for this game's risk class
```

- The manifest is **certified by an independent test lab** (e.g. GLI/eCOGRA-style) and **signed**.
- The kernel **validates every AI intent** against it ([[03-system-architecture]] syscall contract). Out-of-bounds intent → rejected.
- RTP/house edge are **global constants**, not per-player — so "AI personalizes your odds" is impossible (and we say so).

## RNG quality

- Cryptographically secure RNG (CSPRNG) in the kernel; hardware entropy source.
- Certified for statistical randomness as part of lab testing.
- Seed lifecycle logged in the append-only event log for audit.

## What the AI is allowed to do near randomness

To be crystal clear about the boundary:

| AI **may** | AI **may not** |
|---|---|
| Choose *which* certified game/format to run | Change RTP, house edge, or any probability |
| Frame an outcome dramatically | Determine, bias, or reroll an outcome |
| Escalate stake *tiers* within `maxEscalation` | Exceed manifest caps or player RG limits |
| Time/pace when a draw is requested | Peek at future draws |
| Present a near-miss that *actually occurred* | Manufacture near-misses (regulated; [[08-responsible-gambling]]) |

## Dispute resolution

A dispute is settled purely from `(commit, revealed seed, client_seed, event log)` — **recompute and compare.** The AI's narration is irrelevant to the financial truth. This is deterministic, fast, and regulator-friendly ([[09-regulatory-compliance]]).

## Why this ordering matters

Fairness is in the **kernel** specifically so it sits *below* the AI in the trust stack. The most engagement-hungry, jailbroken, hallucinating AI imaginable still gets fair draws out of the kernel and still can't pay anyone a cent the certified manifest doesn't allow. That's the whole game.

→ Next: [[08-responsible-gambling]]
