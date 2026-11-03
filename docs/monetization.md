# Monetization

Status: initial plan, to validate against real customer conversations
before committing engineering effort to billing infrastructure.

## Model

Subscription tiers per organization, plus usage-based AI credits for
heavy AI billing/OCR volume beyond a plan's included quota.

## Proposed tiers

- **Starter (free/low-cost)** — single user, core billing + inventory +
  customer modules, limited AI billing volume per month, no connector
  integrations.
- **Growth** — multi-user with roles, full module set, higher AI volume,
  Tally/connector integration included.
- **Enterprise** — multi-location/multi-warehouse, custom roles, priority
  support, dedicated onboarding, negotiated AI volume.

## Why usage-based AI credits

AI extraction (OCR, LLM calls, speech) has real marginal cost per
transaction, unlike traditional SaaS feature gating. Pricing that ignores
this either underprices heavy users or overprices light users; tying a
portion of pricing to actual AI usage keeps unit economics sound as usage
scales. Revisit self-hosted model economics (see
[ai-architecture.md](ai-architecture.md)) once AI cost becomes a
meaningful share of COGS.

## AI cost model and guardrails

This is the gap that matters most in this document: no real provider
pricing has been checked against real usage patterns yet, so nothing
below is a validated number — it's the framework to run the real numbers
through once pilot data exists, per the validation gate in
[mvp-scope.md](mvp-scope.md).

- **The question that has to be answered before public pricing is set**:
  for a typical pilot business's daily AI-billing volume (OCR + LLM
  structuring per invoice, occasional supplier-PDF price updates), what
  is the actual per-invoice AI cost, and does the intended subscription
  price still leave margin at that volume? Regional competitors
  (Vyapar, Busy) price in the ₹300–1500/**year** range, not per month —
  if AI cost per invoice is even a few rupees and a shop bills 50
  invoices a day, the math has to be checked explicitly, not assumed to
  work because "AI credits" sounds like a solved pricing mechanism.
- **Guardrails that hold regardless of the exact numbers**:
  - Model routing by task difficulty — a cheap, fast model for clean
    typed text or simple photo extraction; a more expensive model
    reserved for genuinely hard cases (poor handwriting, low-quality
    scans), rather than always calling the most capable (and most
    expensive) model available.
  - Caching — a supplier's PDF layout is frequently reused invoice to
    invoice; caching layout-level extraction logic per supplier avoids
    re-paying full OCR/LLM cost on every single document from a known
    source.
  - Per-tier monthly AI-action caps with graceful degradation (falling
    back to manual entry with a clear in-app message) rather than a hard
    paywall interrupting an in-progress invoice.
  - A required internal cost-per-tenant dashboard before any paid
    customer acquisition spend — margin has to be visible before it's
    defended, not discovered after the fact.
- **Explicit gate**: public pricing tiers are not finalized until real
  provider costs are measured against the first pilot cohort's actual
  usage, per [mvp-scope.md](mvp-scope.md)'s validation gate. Everything
  in the "Proposed tiers" section above is a starting hypothesis, not a
  committed price list.

## Go-to-market notes

- Interoperability with Tally (see [integrations.md](integrations.md)) is
  a monetization enabler, not just a technical nicety: it lets Nexora be
  sold as an add-on to existing Tally users without asking them to
  abandon their accountant's workflow, lowering the barrier to a paid
  trial.
