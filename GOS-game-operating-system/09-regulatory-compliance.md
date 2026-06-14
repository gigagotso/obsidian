---
title: Regulatory & Compliance
type: design
tags: [gos, regulation, compliance, licensing, kyc]
created: 2026-06-13
---

# Regulatory & Compliance

Gambling is one of the most heavily regulated industries on earth, and "AI-driven games" are novel enough that regulators will scrutinize hard. The architecture is designed to *answer their questions before they ask*.

> Not legal advice. This is a product-strategy map. Real licensing requires specialist gambling lawyers per jurisdiction.

## The regulator's three questions — and our built-in answers

| They ask | GOS answers with |
|---|---|
| *"Prove a game pays fairly."* | Certified manifest + provable fairness + deterministic replay ([[07-fairness-and-rng]]) |
| *"Prove the AI can't rig or exploit."* | Experience/money separation — AI has no odds/ledger authority ([[03-system-architecture]]) |
| *"Prove you protect vulnerable players."* | Kernel-level RG Guardian, audited, overrides AI ([[08-responsible-gambling]]) |

The whole architecture is essentially a pre-emptive compliance argument. That's deliberate — it's the moat.

## Licensing strategy

- **Start in one well-defined, license-friendly jurisdiction.** Candidates often used by operators: Malta (MGA), Isle of Man, Curaçao (lower bar, lower trust), UK (UKGC — gold standard but strict), specific US states. Pick one and design to its rule set first.
- **Game certification** by an accredited test lab (GLI, eCOGRA, BMM) — they certify the **math manifest** and RNG, *not* the AI narration. Framing the certifiable surface as the deterministic kernel is what makes novel AI games certifiable at all.
- **The AI is positioned as "presentation/UX,"** legally distinct from the "game of chance" (the certified math). This separation isn't just architectural elegance — it's the regulatory wedge that lets an AI-native casino exist.

## KYC / AML (platform services layer)

- **KYC:** identity + age verification before real-money play (gates in the kernel — no bet without verified status).
- **AML:** transaction monitoring, source-of-funds checks, suspicious-activity reporting, sanctions/PEP screening.
- **Geofencing:** enforce per-jurisdiction legality; block where unlicensed.
- **Responsible-gambling reporting:** many regulators require RG interventions to be logged and reported — our event log already captures them ([[03-system-architecture]]).

## Auditability as a first-class feature

Regulators and auditors get:
- **Deterministic replay** of any session from seed + event log.
- **Immutable, append-only logs** of seeds, intents, validations, settlements, RG interventions.
- **Separated AI logs** — the AI's creative choices are logged but explicitly *not* part of the financial record, so audits are clean.

## Novel-tech regulatory risks (be realistic)

- Some regulators may be **uncomfortable with "AI" in gambling** regardless of architecture — expect to *educate* them, possibly via sandbox/pilot programs.
- **Advertising rules** around AI personas and personalization are tightening; marketing must comply ([[08-responsible-gambling]]).
- **Data protection** (GDPR et al.) intersects with the rich [[06-player-model]] — privacy-by-design required.
- Rules differ wildly by jurisdiction; the manifest/profile system should make **per-jurisdiction configuration** (allowed games, limits, messaging) a first-class capability, not a fork.

## Compliance-driven architecture requirements (feed back into design)

1. Per-jurisdiction config layer (games, limits, RG rules, messaging).
2. Verified-status gate in the kernel before any real-money syscall.
3. Append-only audit log with deterministic replay.
4. Independent certifiability of the math surface, isolated from AI.
5. Geofencing + sanctions screening at the platform edge.

→ Next: [[10-economy-and-monetization]]
