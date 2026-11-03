# MVP Scope

This document is the hard line between "building now" and "vision, not
spec yet." Its purpose is to stop the project's own ambition from
becoming its biggest execution risk. Nothing outside this document's V1
scope should be built until the validation gate at the bottom is cleared.

See also [adr/0010-mvp-scope-cut.md](adr/0010-mvp-scope-cut.md) for the
formal decision record behind this scope cut.

## Why this document exists

[PROJECT_SPEC.md](../PROJECT_SPEC.md) describes 12 modules, and
[docs/future-modules.md](future-modules.md) describes a long-term
Business-OS vision spanning CRM, HR, Payroll, Manufacturing, and more.
None of that is wrong as a destination. But this project currently has
zero lines of application code and zero real users. Planning and
architecting for all of it at once is how a small team spends a year
building the "right" system for a product nobody has validated yet.
Zoho — the closest real comparable to the full long-term vision — did not
plan its whole suite up front; it shipped one product, made it work for
real customers, and only then built the next one, over two decades. This
document applies that discipline here.

## V1 scope (first ~90 days, one pilot cohort)

- **Module 1 — Customer Management**: profile, phone/name search, ledger.
  No loyalty program, no advanced segmentation.
- **Module 2 — Inventory**: products, categories, units, single
  warehouse. No batch/expiry, no multi-warehouse, no barcode/QR scanning
  yet.
- **Module 3 — Billing**: GST invoice and retail invoice only. No
  quotations, purchase orders, delivery challans, or recurring invoices
  yet — those are real Module 3 features, just not in the first cohort.
- **Module 4 — AI Billing**: text and photo input only. Voice and
  WhatsApp-forwarded input are Phase 2 (see
  [roadmap.md](roadmap.md)) — text and photo alone already prove or
  disprove the core mechanism.
- **Module 5 — Price Intelligence**: manual upload (PDF/Excel/CSV) →
  diff → approve. No website-scraping ingestion yet.
- **Module 12 — Catalog Intelligence**: the core loop only —
  `candidate_products` staging, live matching, the Product Approval
  Queue, auto-SKU, `price_missing`. The background whole-catalog
  clustering job (see [ai-architecture.md](ai-architecture.md)) is
  deferred until a pilot catalog is actually large enough to need it.
- **GST correctness** (not filing): CGST/SGST/IGST calculation and
  HSN-coded invoices must be correct from the first real invoice — see
  [gst-compliance.md](gst-compliance.md). This is explicitly in V1, not
  Phase 3, because a wrong invoice ends the pilot relationship instantly.

## V1 technology (deliberately smaller than docs/tech-stack.md's target state)

- **One backend service**, not three. Combine `services/core` and the
  V1-relevant parts of `services/ai` into a single deployable unit; split
  them apart later only once there's a real scaling reason to. (Full
  target-state separation remains documented in
  [architecture.md](architecture.md) as where this goes, not where it
  starts.)
- **PostgreSQL only.** No Redis, no Elasticsearch. Postgres trigram
  search (`pg_trgm`) covers V1 fuzzy product matching; revisit dedicated
  search infrastructure only when catalog/query volume actually demands
  it.
- **Web app only.** No `apps/mobile` in V1 — a responsive web app is
  sufficient to validate the core loop with a pilot cohort before paying
  the cost of a native app and a real offline-sync engine (see
  [offline-strategy.md](offline-strategy.md)).
- **No Tally local agent in V1.** Tally-using pilot businesses are
  bridged with manual CSV export/import as a stopgap while the local
  agent is prototyped in parallel against 2-3 real Tally installs (see
  [integrations.md](integrations.md) and
  [adr/0012-tally-adoption-sequencing.md](adr/0012-tally-adoption-sequencing.md)).
  Do not commit to building the full agent until that prototype confirms
  it's feasible to install and support.
- **No Kubernetes.** A single managed-container deployment is enough for
  a pilot cohort — see [deployment.md](deployment.md).
- **English input only for structured UI text**; AI input language
  support is addressed on its own timeline — see
  [roadmap.md](roadmap.md) and [ai-architecture.md](ai-architecture.md).

## Explicitly out of scope for V1

CRM, HR, Payroll, Manufacturing, IoT, POS, e-commerce/marketplace
integration, full accounting/GST filing, multi-branch management — all of
these are real, named in
[docs/future-modules.md](future-modules.md), and none of them get a PRD
or an architecture decision until V1 is validated with real paying or
committed-pilot customers.

## Validation gate before Phase 2 begins

Do not start Phase 2 ([roadmap.md](roadmap.md)) until all of the
following are true:

1. At least 5-10 real businesses have used V1 for real billing, not a
   demo, for at least two consecutive weeks.
2. Catalog Intelligence (Module 12) has been exercised against real,
   messy pilot data (actual handwritten lists, actual supplier PDFs) —
   not just the fixture set in [testing-strategy.md](testing-strategy.md)
   — and the confidence thresholds have been recalibrated against what
   actually happened, not left at their initial placeholder values.
3. At least one real GST invoice generated by Nexora has been accepted
   without correction by the business's own accountant.
4. Real AI provider costs per invoice have been measured against pilot
   usage and checked against the cost model in
   [monetization.md](monetization.md) — pricing is not finalized until
   this is done.
5. The connectivity assumption in
   [offline-strategy.md](offline-strategy.md) has been checked against
   how often pilot businesses actually lose connectivity mid-billing.

Any of these failing is a reason to change V1, not a reason to skip ahead
to Phase 2 anyway.
