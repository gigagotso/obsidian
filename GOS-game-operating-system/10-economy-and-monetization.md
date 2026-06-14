---
title: Economy & Monetization
type: business
tags: [gos, business, economy, monetization]
created: 2026-06-13
---

# Economy & Monetization

How GOS makes money — and the cost structure that determines whether the whole thesis survives contact with reality.

## Primary revenue: the house edge (GGR)

Like any casino, the core revenue is **Gross Gaming Revenue** = total wagered × house edge. The edge is in the certified manifest ([[07-fairness-and-rng]]), global and fixed.

- Typical online house edge: ~1–5% depending on game class.
- GGR scales with **volume × edge × retention**.
- GOS's bet: **better experience → higher healthy retention & session value** at the *same* edge — i.e. we win on engagement quality, not by raising the edge (and never by exploiting players — [[08-responsible-gambling]]).

## The cost that could kill it: AI per session

This is the existential number. Unlike a static slot (near-zero marginal cost per spin), GOS spends **inference** per session.

```
Per-session margin = (avg wagered × house edge)  −  AI inference cost  −  platform cost
                     ─────────────────────────────   ───────────────────   ────────────
                          must stay well > 0
```

If a session's AI cost approaches its GGR contribution, the model breaks. Mitigations (designed into [[04-ai-game-engine]]):
- **Tiered models:** cheap/fast model for frequent beats, strong model only for arc decisions.
- **Heavy pre-generation & caching** of common beats; live model fills gaps.
- **Budget caps per session** with scripted fallback.
- **Scale AI spend to player value:** high-roller sessions justify richer AI; low-stakes sessions run leaner/scripted.

→ This ratio is the #1 thing to validate early ([[13-risks-and-open-questions]], [[12-roadmap]] Phase 0).

## Why higher engagement is worth real AI cost

The bet is that AI-directed sessions raise **healthy** lifetime value via:
- longer *enjoyable* (not compulsive) sessions,
- higher return rate (the rival/story pulls you back),
- premium positioning vs. commodity slots,
- word-of-mouth from a genuinely novel experience.

The discipline: grow LTV through *delight and retention*, never through harm. A player who churns out broke is a worse business than one who plays for years for fun ([[08-responsible-gambling]] "safety as product").

## Secondary / future revenue

- **Platform/B2B licensing.** GOS is an *operating system* — license the engine to other operators who ship their own game manifests + personas. Possibly the bigger long-term business than running a casino ourselves: be the "Unity/Stripe of AI gambling," sell picks-and-shovels.
- **Premium personas / cosmetic content** (non-pay-to-win; cannot affect odds — Pillar 1).
- **Tournaments / events / season passes** (entertainment layer, [[05-game-modes-catalog]] C3).
- **Branded/themed experiences** (IP partnerships skinned onto certified cores).

## On tokens / crypto (proceed with caution)

Crypto rails fit provable fairness culturally and ease some payments, **but**:
- They invite the unregulated-casino reputation we explicitly reject ([[01-vision-and-concept]]).
- Regulatory/AML scrutiny is heavier, not lighter.
- **Recommendation:** treat crypto as an *optional payment rail* under full KYC/AML in jurisdictions that allow it — **not** as the brand identity or a "we don't need a license" shortcut. The provable-fairness *math* is the valuable idea; it does not require a token.

## Unit-economics scorecard (track from day one)

- GGR per active player
- AI inference cost per session / per $ wagered
- Healthy retention (return rate of *non-harmed* players)
- Session length & enjoyment (vs. compulsion signals)
- RG intervention rate (a *health* metric, watch it goes the right way)
- LTV : CAC

→ Next: [[11-tech-stack]]
