---
title: Roadmap
type: plan
tags: [gos, roadmap, phases, plan]
created: 2026-06-13
---

# Roadmap

Phased so that the **riskiest assumptions are tested cheapest and first**. The order is deliberate: don't build the casino before proving people want the experience and the economics work.

## Phase 0 — Validate the two existential assumptions (weeks, play-money)

**Goal:** before any regulator, ledger, or real money, answer two questions:
1. *Is an AI-directed gambling session meaningfully more engaging?* (the product bet)
2. *Can per-session AI cost stay well below its value?* (the economics bet — [[10-economy-and-monetization]])

**Build:** a throwaway, **play-money** prototype of **A1 The Duelist** ([[05-game-modes-catalog]]) — fair-ish RNG, one AI rival persona with memory, escalation, banter. No kernel rigor yet.

**Measure:** session length, return rate, "would you play this with real money?", and **inference cost per session**.

**Kill criteria:** if play-money users shrug, or AI cost ≈ plausible GGR, stop and rethink. Cheap to learn this now.

## Phase 1 — The Trust Kernel (the moat)

**Goal:** build the part regulators and players actually need ([[03-system-architecture]], [[07-fairness-and-rng]]).

**Build:**
- Provably-fair RNG (commit-reveal) + player-side verify tool.
- Double-entry append-only ledger + deterministic replay.
- Signed **game manifest** + intent validator (the syscall waist).
- Kernel **RG Guardian** with pre-bet veto ([[08-responsible-gambling]]).
- Re-implement A1 *properly* on top of the kernel.

**Exit:** a single game that is provably fair, replayable, RG-guarded, and whose AI provably cannot touch money/odds. This is the demo that unlocks everything else.

## Phase 2 — The Flagship Experience

**Goal:** prove the "wow," not just the plumbing.

**Build:**
- **B1 Heist Night** ([[05-game-modes-catalog]]) — full AI Director arc, live theming, the trailer moment.
- The primitive grid ([[04-ai-game-engine]]): a few formats × themes × personas × arcs → many sessions from few cores.
- Player model v1 ([[06-player-model]]) with cross-session narrative memory.
- Latency engineering: pre-gen, streaming, scripted fallback (Pillar 8).

**Exit:** something that makes people say "I've never seen a casino do *that*."

## Phase 3 — Compliance & Real Money (one jurisdiction)

**Goal:** go live, legally, somewhere small and friendly ([[09-regulatory-compliance]]).

**Build:**
- KYC/AML, geofencing, per-jurisdiction config.
- Independent lab **certification** of the math manifest + RNG.
- Full RG suite + reporting; transparency dashboards.
- Payments (and *optionally* a KYC'd crypto rail).

**Exit:** licensed, certified, live with real money in one jurisdiction. Learn the real unit economics.

## Phase 4 — Scale the Catalog & the Player

**Goal:** breadth and depth.

**Build:**
- More game modes (Tier B, early Tier C — [[05-game-modes-catalog]]).
- Richer player model & personalization (within the ethical lines).
- Tournaments / live ops / social co-op.
- Tighten AI cost via better tiering/caching.

## Phase 5 — GOS as a Platform (the bigger business)

**Goal:** become the **operating system**, not just one operator ([[10-economy-and-monetization]]).

**Build:**
- Open the manifest + persona system so **third-party studios** ship games onto GOS.
- B2B licensing of the engine to other operators.
- Multi-jurisdiction expansion via the config layer.
- Be the picks-and-shovels layer for AI-native gambling.

## The throughline

> Each phase de-risks the next. Phase 0 proves *want*. Phase 1 proves *trust*. Phase 2 proves *wow*. Phase 3 proves *legal & economics*. Phases 4–5 prove *scale*. Don't skip ahead — most AI-casino ideas die by building Phase 2 flash before Phase 0/1 substance.

→ Next: [[13-risks-and-open-questions]]
