---
title: Game-Type Taxonomy — AI primitives as betting mechanics
type: design
tags: [gos, unique, games, taxonomy]
created: 2026-06-13
---

# Game-Type Taxonomy — AI primitives as betting mechanics

The generative way to invent AI-native games: don't list games, list the **AI capabilities** that can be turned into a *source of fair uncertainty*, then spawn games from each. Every type below passes the **delete-the-AI test** — remove the AI and the game ceases to exist (vs. a skin, where it survives unchanged → [[14-the-unique-thesis]]).

For each primitive, the critical question is **"where's the house edge, and can the [[14-the-unique-thesis|Odds Solver]] price it?"** A game with no defensible, computable edge can't ship.

---

## A. Prediction — *AI models a person*
**Capability:** model someone's decisions and predict them.
**Edge source:** humans are ~60–70% predictable (Aaronson oracle). The gap above 50% is the edge.

- **Oracle** — defy the machine's locked prediction of *you*. ([[15-flagship-oracle]])
- **Mentalist** — *you* predict an AI persona (or another player). Inverts who's modeled; rewards theory-of-mind. Edge from your imperfect model of *them*.
- **Crowd-beat** — beat the predictor trained on the whole population. "Are you more random than the average human?"

🟢 Most novel, cheapest to build, philosophically loaded. ⚠️ Behavioral model must stay walled in-game ([[15-flagship-oracle]] ethics).

---

## B. Generation-judged — *AI evaluates open-ended human output*
**Capability:** score creative/persuasive output on a calibrated scale.
**Edge source:** the judge's difficulty threshold is tuned so win-probability is known and < 50% net of payout.

- **The Pitch** — persuade an AI judge to award you the pot in N messages. Rhetoric is the game.
- **Wordsmith / One-Liner** — produce the better line vs. an AI bar; AI scores. Skill+chance.
- **Move the Needle** — make a measurable AI reaction cross a threshold (make it concede, make it rate you ≥ X).

🟡 Skill-heavy → regulatory grey zone (game of skill vs chance varies by jurisdiction, [[09-regulatory-compliance]]). Edge must be airtight & the judge un-gameable/un-injectable.

---

## C. Semantic-space — *AI reasons in meaning, not symbols*
**Capability:** embeddings + language understanding → distance in concept-space is measurable.
**Edge source:** the hidden target's reachability is solvable; payout scales with proximity achieved.

- **Closer** — get as near a hidden concept as you can in K guesses (semantic "warmer/colder").
- **Taboo Vault** — make the AI *say* the secret word without using trigger words. Coax, don't state.
- **Driftwood** — a word-chain that must drift toward a secret target without overshooting; bust if you land on a banned neighbor.

🟢 Genuinely new feel — gambling in meaning-space. Needs live embeddings (cheap). Great for [[14-the-unique-thesis|Forge]] primitives.

---

## D. Persuasion / adversarial — *talk the AI into something*
**Capability:** an AI keeper with a goal you must subvert through dialogue.
**Edge source:** the keeper's resistance is tunable; the Solver prices "crack probability" per difficulty tier.

- **The Vault** — negotiate/persuade an AI keeper to hand over the pot before your message budget runs out.
- **Confession** — extract a hidden secret (a number, a fact) under a question budget. Twenty-questions vs. an adversarial keeper.
- *Sanctioned, bounded, FAIR "jailbreak-as-sport"* — the thing players already try for free, turned into a priced game with a real prize.

🟡 Must be injection-hardened (the keeper *is* the attack surface). The "weird narrative, never theft" guarantee ([[03-system-architecture]]) is what makes this safe to run.

---

## E. Emergent simulation — *bet on AI-driven worlds*
**Capability:** multi-agent simulations produce outcomes no one pre-computed.
**Edge source:** the Solver runs the sim thousands of times offline → true odds → sets edge. Bettor sees indicative odds, not the seed.

- **Coliseum** — AI agents/creatures battle or race; emergent winner. Sabermetrics for fictional worlds.
- **Petri** — an evolving ecosystem/economy; bet on where it lands (which species/faction dominates).
- **Civics** — AI civilizations rise/fall/elect; bet on the arc.

🟢 This is the purest expression of "bet on generative outcomes" and "the game unfolds." Visually dramatic, very demoable in pure code. **Best candidate for the next playable demo.**

---

## F. Social deduction — *AI personas with hidden agendas*
**Capability:** LLM agents that bluff, lie, ally, betray — convincingly.
**Edge source:** reading deception is hard but learnable; edge from the agents' calibrated bluff rates.

- **Table of Liars** — poker-of-personalities: read/out-bluff a table of AI characters with secret motives.
- **Impostor** — which of N agents is lying / sabotaging? Werewolf with priced stakes.

🟡 Rich and sticky; expensive (many live agents per table). Pairs with persona memory ([[06-player-model]]).

---

## G. Discrimination / forgery — *fool or serve a classifier*
**Capability:** AI can both generate and detect.
**Edge source:** the discriminator's accuracy sets the odds (a GAN equilibrium).

- **Turing Table** — spot the human among the bots (or *pass as a bot* yourself). Bet on detection.
- **Forger vs Critic** — produce output an AI critic accepts as genuine X; the critic's strictness is the edge.

🟢 Conceptually crisp, screenshot-friendly, ties to a real ML idea players find fascinating.

---

## H. Open proposition market — *AI prices anything you can say*
**Capability:** turn arbitrary natural-language propositions into priced markets in real time.
**Edge source:** AI market-maker quotes with a built-in spread; Solver bounds tail risk; exposure caps per prop.

- **Say Anything** — propose *any* proposition in plain language ("rain in Tbilisi before my coffee", "I can name 10 capitals in 20s") → instant fair odds → bet. Infinite betting surface, no catalog at all.

🔴 Highest-variance & hardest to govern (real-world props = oracle/settlement + integrity problems). Powerful but a later-phase moonshot. Heavy overlap with the Solver risk ([[13-risks-and-open-questions]] R10).

---

## How this feeds the platform

- Each primitive is a **composable block** the [[14-the-unique-thesis|Forge]] can drop into generated games (e.g. "a *Coliseum* where you can *persuade* one fighter mid-match").
- Each needs the **Odds Solver** to compute and certify its edge — that's the unifying kernel dependency.
- Each still obeys **Pillar 1**: the AI runs the *mechanic*, the kernel/solver owns the *edge* ([[02-core-pillars]]).

## Quick scorecard

| Type | Novelty | Build cost | Edge clarity | RG/regulatory risk |
|---|---|---|---|---|
| A Prediction | ★★★ | low | high | model-leak risk |
| B Gen-judged | ★★ | med | medium | skill-vs-chance grey |
| C Semantic | ★★★ | low-med | medium | low |
| D Persuasion | ★★★ | med | medium | injection surface |
| E Emergent sim | ★★★ | med | high | low |
| F Social deduction | ★★ | high | medium | low |
| G Discrimination | ★★ | med | high | low |
| H Open prop market | ★★★ | high | low | high |

**Build-next recommendation:** **E. Emergent simulation (Coliseum)** — high novelty, clean computable edge, low RG risk, and the most literal demo of "the game unfolds dynamically." Then **C. Semantic** for a second, very different feel.

→ Thesis: [[14-the-unique-thesis]] · Signature: [[15-flagship-oracle]] · Back to [[00-INDEX]]
