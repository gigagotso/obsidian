---
title: Risks & Open Questions
type: analysis
tags: [gos, risks, questions]
created: 2026-06-13
---

# Risks & Open Questions

The honest list. A plan without this is marketing. Each item is something that could break the thesis or is genuinely unresolved.

## Existential risks (could kill the whole thing)

### R1 — AI cost vs. margin
The model dies if per-session inference cost approaches per-session GGR ([[10-economy-and-monetization]]). **Unproven.** → Test in [[12-roadmap]] Phase 0 before anything else.

### R2 — Engagement uplift may not be real
"AI-directed gambling is more engaging" is a hypothesis, not a fact. Slot players may not want story; they may want fast, mindless spins. → Phase 0 play-money test must show real uplift.

### R3 — Regulators may reject "AI in gambling" categorically
Even with perfect architecture, some regulators may balk at the word "AI." → Pick a sandbox-friendly jurisdiction; lead with the experience/money separation as the educational hook ([[09-regulatory-compliance]]). Risk that *no* good jurisdiction bites early.

### R10 — The Odds Solver is safety-critical (unique-direction risk)
The [[14-the-unique-thesis|wager-compiler]] direction lives or dies on the Odds Solver correctly pricing **never-before-seen, AI- or player-generated** games. A mis-priced game = unbounded house liability *or* a free edge for sharps to farm. → Needs confidence bounds, a "refuse to certify if uncertainty too high" failsafe, exposure caps per generated game, and adversarial testing against players who design games to break the solver. New, and serious.

### R4 — The ethical tension may not be reconcilable
An adaptive, persuasive engagement engine on a gambling product is in real, possibly irreducible tension with player welfare ([[08-responsible-gambling]]). Architecture mitigates but does not *erase* it. **Open question we should keep asking out loud:** can this be done such that harmed-player rates genuinely go *down* vs. incumbents? If, after honest measurement, the answer is no, that is a reason not to build it — and that has to be a real option on the table, not a rhetorical one.

## Serious risks (could badly wound it)

### R5 — Latency breaks the feel
If live AI can't hit the beat-latency budget, tables feel broken (Pillar 8). → Pre-gen/caching/fallback must actually work at scale ([[11-tech-stack]]). Unvalidated under load.

### R6 — Prompt injection / jailbreak via player chat
Players *will* try to jailbreak the dealer. Architecture says worst case = weird narrative, never theft ([[03-system-architecture]]). → Must red-team hard; a viral "I made the AI casino say X" moment is a brand risk even if no money moves.

### R7 — The novelty wears off
Generated content can still feel samey once players see the seams of the primitive grid ([[04-ai-game-engine]]). → Need genuine variety + fresh cores, or churn spikes after the wow.

### R8 — Trust paradox
"AI gambling" sounds *less* trustworthy to the average person, even though GOS is *more* verifiable. → Provable fairness must be front-and-center marketing, not a buried feature ([[07-fairness-and-rng]]). Education cost is real.

### R9 — Reputational contagion
The AI-gambling/crypto-casino space is full of scams. Being adjacent risks guilt by association. → Aggressive licensing + transparency to differentiate; avoid the crypto-casino aesthetic ([[10-economy-and-monetization]]).

## Open design questions (unresolved, need decisions)

- **Q1.** Where exactly is the line between "AI frames an outcome" and "AI manipulates"? Near-miss *framing* is the sharp edge ([[08-responsible-gambling]]). Needs concrete, testable rules + regulator sign-off.
- **Q2.** How much cross-session memory is "magical rival" vs. "creepy surveillance"? ([[06-player-model]]) Where's the player-comfort line?
- **Q3.** PvAI formats: how do we make an AI opponent feel skilled/alive while *guaranteeing* it has no informational/odds edge — and prove that to players? ([[05-game-modes-catalog]] B2)
- **Q4.** First jurisdiction: trustworthy-but-strict (UKGC/MGA) vs. fast-but-cheap (Curaçao)? Trades speed for brand. ([[09-regulatory-compliance]])
- **Q5.** Casino-operator first, or platform/B2B first? The OS framing suggests platform is the bigger prize, but operating ourselves proves the product. ([[10-economy-and-monetization]] Phase 5 vs earlier)
- **Q6.** Crypto rail: in or out for v1? Cultural fit with provable fairness vs. reputational/regulatory drag. (Recommendation leans *out* for v1.)
- **Q7.** How do we *measure* "healthy vs. compulsive" engagement well enough to optimize for the former? The whole ethical model depends on this metric existing and being honest.

## What would make me more confident

1. Phase 0 data: real engagement uplift **and** workable AI cost (R1, R2).
2. A regulator (or credible advisor) confirming the experience/money split is certifiable (R3).
3. A measurable, declining harmed-player rate vs. incumbents (R4, Q7).
4. A working latency budget under realistic load (R5).

Until those four exist, GOS is a compelling thesis, not a validated business. That's fine — that's exactly what [[12-roadmap]] Phase 0–1 is for.

← Back to [[00-INDEX]]
