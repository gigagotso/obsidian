---
title: Game Modes Catalog
type: design
tags: [gos, games, concepts]
created: 2026-06-13
---

# Game Modes Catalog

Concrete game concepts that showcase "AI plays with you, game unfolds." Each is one certified math core ([[07-fairness-and-rng]]) wrapped in dynamic AI direction ([[04-ai-game-engine]]). Ordered roughly easiest → most ambitious to build.

> Reading note on every entry below: wherever the AI "reacts," "reads," "bluffs," or "escalates," the underlying odds are fixed and provably fair. The AI moves the *story*, never the *math* (Pillar 1 & 7, [[02-core-pillars]]).

---

## Tier A — buildable for MVP (single certified core, AI flavor on top)

### A1. The Duelist (AI rival, dice/coin core)
A best-of-N push-your-luck duel. The AI is a **named rival with memory**.
- Core math: simple over/under or coin-style draw, fixed edge.
- AI layer: taunts, escalates the wager ("double or nothing — unless you're scared"), references your last visit, builds a rivalry arc.
- *Unfolds:* the rival's mood, the story stakes, and the escalation ladder are generated; the dice are fair.
- **Why first:** trivial math, maximum "wow" from the persona. Best demo of the thesis.

### A2. Crash, Narrated (multiplier curve core)
The classic rising-multiplier "cash out before it crashes" game, but the AI **narrates the climb** and plays a parallel "ghost" that cashed out at some point — taunting or cheering you toward the edge.
- Core math: certified crash curve distribution.
- AI layer: tension narration, the ghost rival, dynamic theming (a rocket / a heist getaway / a stock bubble).
- *Unfolds:* theme + rival behavior; **never** the crash point (kernel-drawn, provably fair).
- ⚠️ Tilt-sensitive format → strong RG hooks required ([[08-responsible-gambling]]).

### A3. The Ladder (push-your-luck, themed)
Climb tiers; each step risks it all for more. AI is a **guide/tempter**.
- Core math: per-step success probability table, certified.
- AI layer: a mentor who warns, or a devil who tempts; the climb is reframed as a dungeon descent / mountain / heist floors.
- *Unfolds:* narrative pressure to continue vs. the guardian nudging you to bank.

---

## Tier B — flagship (richer interaction, still one core each)

### B1. Heist Night (narrative roulette/wheel core)
A full **procedurally-generated heist** where each "phase" is a wheel/draw, and the AI Director builds a one-session story: the crew, the vault, the betrayal, the escape.
- Core math: a sequence of independent certified draws with published odds.
- AI layer: the heist plot is generated live; outcomes are narrated as plot beats ("the laser grid — will you slip through?").
- *Unfolds:* the entire story; replayable for a *different* heist every time.
- This is the **signature experience** — the trailer.

### B2. The Read (AI "reads you" — card draw core)
A bluff-flavored card game where the AI **claims to read your tells** (your bet sizing, your chat, your timing). It's theatre over a fair draw, and *we say so transparently.*
- Core math: certified card/number draw, fixed edge.
- AI layer: a mentalist persona that comments on your patterns, sometimes "right," sometimes "wrong" — all flavor.
- **Trust play:** lean INTO transparency. "The AI's reads are for fun; the cards are provably fair — verify any hand." Turns a creepy idea into a trust feature (Pillar 7).

### B3. Boss Gauntlet (escalating PvAI series)
A run of escalating "boss" opponents, each a distinct persona with its own theme and a tougher *story* (not tougher odds). Roguelike meta-structure across a session.
- Core math: per-boss certified core (can reuse cores with new skins).
- AI layer: bosses with arcs, banter, and callbacks across your run.
- *Unfolds:* which bosses, in what order, with what narrative — generated per run.

---

## Tier C — ambitious / future (multi-core, social, agentic)

### C1. The Living Casino (open-world session)
You "walk" a generated casino floor; the AI populates it with rivals, side-bets, events. A meta-OS where many Tier A/B games are rooms.
- The Director becomes a *world* director. Persistent NPCs with memory across visits.

### C2. Co-op vs. the House (social)
Several real players + an AI "captain" coordinate against a house scenario the Director runs. Shared narrative, individual fair wallets.
- Hard: social dynamics, shared state, but huge engagement/virality.

### C3. Tournament Storylines (live ops)
Time-boxed events where the AI weaves all participants into one evolving meta-narrative (a season). Leaderboards as story beats.

---

## Cross-cutting design rules for every mode

1. **One certified math core, many skins.** Never invent new odds for a new theme — re-skin a certified core ([[07-fairness-and-rng]]).
2. **The AI's "decisions" are theatre or in-rules.** Document for each mode exactly *what the AI can and cannot influence* (regulator + player trust).
3. **RG hooks per mode.** Crash/ladder/push-luck are the riskiest; they get the strongest interventions ([[08-responsible-gambling]]).
4. **Verifiable after the fact.** Every mode exposes its seed-reveal so a player can audit ([[07-fairness-and-rng]]).
5. **Graceful degradation.** Every mode must still be playable with the AI in scripted-fallback mode (Pillar 8).

## Build priority for a demo

> **A1 The Duelist** → proves the thesis with the least code. Then **B1 Heist Night** as the flagship trailer. Everything else after validation. See [[12-roadmap]].

→ Next: [[06-player-model]]
