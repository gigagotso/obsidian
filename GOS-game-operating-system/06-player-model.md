---
title: Player Model
type: design
tags: [gos, player, personalization, ethics]
created: 2026-06-13
---

# Player Model

The living model of each player that makes the AI feel like it *knows* you — and the same data the RG Guardian uses to keep you safe. This file is where personalization and ethics collide head-on.

## What it stores

### Identity & state (kernel-trusted)
- Verified identity (KYC), jurisdiction, account status ([[09-regulatory-compliance]]).
- Wallet balance, transaction history (ledger).
- **RG state:** self-set limits, cool-off/exclusion status, affordability signals. *Read by the kernel guardian, authoritative.* ([[08-responsible-gambling]])

### Preference & taste (AI-readable)
- Favorite formats, themes, persona types.
- Pacing preference (slow-burn vs. fast).
- Narrative callbacks: rivalries, past "boss" defeats, running jokes.
- Stylistic likes (humor, intensity, lore depth).

### Session signals (ephemeral, derived)
- Tilt indicators: bet-size acceleration, loss-chasing patterns, session length, time-of-day risk.
- Engagement: are they enjoying it or grinding?
- Sentiment from chat.

## Two consumers, opposite goals — by design

The same player model feeds two systems with **deliberately opposed objectives**:

| Consumer | Wants | Reads |
|---|---|---|
| **AI Director** ([[04-ai-game-engine]]) | a more *engaging* session | taste + session signals |
| **RG Guardian** ([[08-responsible-gambling]]) | a *safe* session | RG state + tilt signals |

This tension is **intentional and protective**. The Guardian sits in the kernel and **wins ties** — if the tilt signal says "this person is chasing losses," the Guardian's cool-down overrides the Director's "escalate the rivalry." Engagement is allowed to push right up to, but never through, the welfare boundary (Pillar 3, [[02-core-pillars]]).

## The central ethical line

> Personalization may be used to make the game **more fun**. It may **never** be used to make a player **bet more than they safely can**.

Concretely, this means a hard list of **forbidden optimizations**:
- ❌ No targeting known-vulnerable players with higher-intensity content.
- ❌ No using tilt/desperation signals to *increase* stakes prompts.
- ❌ No personalized "win-back" pressure after self-exclusion.
- ❌ No dynamic pricing/odds by player (odds are global & certified anyway — Pillar 1).
- ❌ No dark-pattern friction asymmetry (easy to deposit, hard to withdraw).

These are enforced as **policy checks in the kernel**, audited, and logged — not left to the AI's discretion. Crossing one should be *structurally impossible*, the way the AI can't touch the ledger.

## Data governance

- **Minimization:** store the least needed for personalization + safety.
- **Player visibility:** a player can see what the model "thinks" of them (taste profile) — transparency as trust.
- **Consent & control:** opt-out of personalization (game still works, just less tailored).
- **Privacy/jurisdiction:** GDPR-style rights; data residency per [[09-regulatory-compliance]].
- **Separation:** RG/affordability data is *especially* protected and never used for marketing.

## Memory across sessions (the "it remembers me" magic)

The biggest emotional payoff — a rival who remembers beating you, a mentor who recalls your comeback — comes from **persistent narrative memory**. Keep this layer:
- **Curated:** store narrative facts ("beat the Trickster boss on 3rd try"), not raw surveillance.
- **Bounded:** memory feeds flavor, never odds or pressure.
- **Deletable:** player can wipe their "story" without losing their wallet.

## Cold start

New player with no history → the AI uses safe defaults (medium pace, friendly persona, conservative RG posture until affordability is established). Personalization *earns* its way in over sessions.

→ Next: [[07-fairness-and-rng]]
