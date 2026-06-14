---
title: AI Game Engine
type: design
tags: [gos, ai, agents, engine]
created: 2026-06-13
---

# AI Game Engine

How a game actually *unfolds dynamically*. This is the userland from [[03-system-architecture]] — the creative brain that makes each session alive, bounded by the kernel.

## The agent ensemble

Rather than one monolithic prompt, GOS runs a small ensemble of specialized agents per session. Each is cheap, focused, and individually replaceable.

### 1. Game Director (the "showrunner")
Owns the **arc** of the session.
- Chooses the starting format + theme from the manifest's allowed set.
- Decides when to escalate, twist, introduce a bonus round, or wind down.
- Maintains the **dramatic curve** (setup → rising tension → climax → resolution).
- Emits high-level intents: `escalate(tier)`, `introduceTwist(id)`, `proposeBonus(id)`.
- Thinks in *beats*, not turns.

### 2. Dealer / Opponent (the "face")
The persona at the table.
- Generates the moment-to-moment dialogue, taunts, reactions, coaching.
- In PvAI formats, makes in-rules decisions (when allowed to "play").
- Carries persona + memory of the player ([[06-player-model]]).
- **Constraint:** its decisions are theatre or within published rules — never odds/secret-info (Pillar 7, [[02-core-pillars]]).

### 3. Narrative / Theme Engine
- Renders the world: a heist, a duel at dawn, a cosmic casino, a noir back-room.
- Can **re-skin live** ("the room shifts as you double down").
- Keeps continuity across a session and across sessions (callbacks: "back for revenge?").

### 4. Pacing / Difficulty agent
- Watches tempo, tilt, and engagement signals.
- Adjusts *structure* (round length, choice frequency, tension), not odds.
- Feeds the RG Guardian: flags acceleration that looks like chasing losses ([[08-responsible-gambling]]).

### 5. RG Guardian liaison (userland side)
- The userland shadow of the kernel guardian.
- Softens/reshapes the experience toward safety *before* the kernel has to hard-veto (offer a break, cool the narrative down). Belt-and-suspenders.

## The session loop

```
bootstrap:
  kernel commits seed (provably fair) ─ player sees commit hash
  Director loads manifest, picks format + theme + persona
  Narrative sets the opening scene

loop (per beat):
  1. Player acts (bet / choice / chat)
  2. RG Guardian (kernel) checks limits ───────► may veto/end
  3. Director decides next beat → emits intent
  4. Kernel validates intent vs manifest ──────► may reject
  5. Kernel performs draw / settles (syscalls)  ◄── provably fair outcome
  6. Dealer + Narrative render the outcome dramatically
  7. Pacing agent updates tempo, feeds player model
  until Director resolves OR limits hit OR player leaves

teardown:
  kernel reveals seed → player can verify every draw
  session logged for replay/audit
```

The crucial line is **step 4–5**: the AI says "deal a tense near-miss" but it *cannot* manufacture a near-miss — it requests a draw, the kernel returns a fair result, and the AI only gets to *frame* whatever actually came up. (A near-miss can only be *presented*, never *engineered* — and even presentation of near-misses is regulated; see [[08-responsible-gambling]].)

## Where "dynamic unfolding" comes from — the primitive grid

Content = combinations of certified primitives. The AI navigates this grid live:

```
FORMATS  ×  THEMES        ×  PERSONAS      ×  ARC SHAPES
─────────   ───────────      ───────────     ──────────
dice duel   heist           cocky rival     slow burn
card draw   noir            zen croupier    fast & loud
wheel       cosmic          trickster       comeback story
ladder/     mythic          mentor          boss fight
push-luck   cyberpunk       silent assassin gauntlet
crash curve western         ...             ...
```

A handful of each → thousands of distinct, coherent session experiences from one certified math library. *This* is the scalability thesis (Pillar 4).

## Bounding the AI (safety mechanics)

- **Intent allowlist:** the AI can only emit intents the manifest declares. Free-form actions are impossible by construction.
- **Schema-validated output:** every agent returns structured intents (JSON schema), not prose the kernel must parse. Prose is *only* for the player-facing channel.
- **Prompt-injection assumption:** treat all player chat as hostile input. The AI userland having *zero* money/odds authority means a successful injection yields, at worst, weird narrative — never theft.
- **Budgeted + cached:** common beats are pre-generated; the live model fills gaps. Latency budget per beat is hard-capped, with scripted fallback (Pillar 8).
- **Determinism for replay:** AI creative choices are logged as events so a session can be *narratively* replayed too, but financial replay never depends on them ([[03-system-architecture]]).

## Model strategy (sketch)

- **Cheap/fast model** for high-frequency beats (taunts, reactions) — latency-critical.
- **Stronger model** for arc decisions (Director) — runs less often, can think.
- **Tiny classifier** for RG/tilt signals — runs every beat, near-instant.
- Heavy use of pre-generation + caching to keep per-session AI cost below per-session margin ([[10-economy-and-monetization]]). This cost ratio is an existential assumption → [[13-risks-and-open-questions]].

→ Next: [[05-game-modes-catalog]]
