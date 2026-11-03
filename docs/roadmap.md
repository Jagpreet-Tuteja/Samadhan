# Roadmap

## Phase 0 — Validation (before Phase 1 work is considered "done")

Phase 1 is not complete just because the code ships — it's complete when
the five-point validation gate in [mvp-scope.md](mvp-scope.md) passes: 
real pilot businesses using it for real billing, Catalog Intelligence
tested against real messy data with recalibrated thresholds, a real GST
invoice accepted by a real accountant, real AI cost-per-invoice measured
against pricing, and the offline/connectivity assumption checked against
reality. Phase 2 does not start until this gate passes, regardless of
how much of Phase 1's feature list is technically built.

## Phase 1 — Core Operations
- Billing (Module 3), including correct GST tax-split, HSN codes, and
  composition-scheme handling on every invoice from day one — see
  [docs/gst-compliance.md](gst-compliance.md). GST *filing* (GSTR-1/3B,
  e-invoicing, e-way bills) stays in Phase 3; GST *correctness* on
  invoices does not wait that long. V1 scope done 2026-11-03 (tax
  invoices and bills of supply; invoice PDF, quotations, purchase and
  credit/debit documents still to come).
- Inventory (Module 2) — V1 scope done 2026-11-03 (with manual retail
  pricing, auto-SKU and `price_missing`).
- Catalog Intelligence, core loop only: `candidate_products` staging,
  live fuzzy+attribute matching, the Product Approval Queue, auto-SKU,
  and `price_missing` (Module 12 — see
  [docs/product-catalog-lifecycle.md](product-catalog-lifecycle.md)).
  This has to land alongside Billing and Inventory, not after them —
  every other AI module depends on products being creatable without a
  pre-existing catalog.
- Customer management (Module 1) — done 2026-09-13.
- Supplier management (Module 7)
- Expense management (Module 8)
- Business dashboard (Module 9)
- AI Business Assistant, text-only (Module 10, partial)

## Phase 2 — Reach & Capture
- WhatsApp Business integration (input + output channel)
- Barcode/QR scanner
- Mobile app (`apps/mobile`)
- Web dashboard polish
- Role management (full RBAC, [docs/permissions.md](permissions.md))
- Notifications (low stock, payment due, price-update pending approval)
- AI Billing via voice and photo (Module 4)
- Hindi and Hinglish (code-mixed) input support for AI Billing and OCR —
  moved up from an unscheduled backlog item. A meaningful share of this
  market's handwritten notes, voice input, and WhatsApp messages are not
  pure English; English-only input risks not working at all for a real
  share of target users, not just being a polish gap. Scope for real
  validation as part of the Phase 0 gate before this phase is called
  complete.

## Phase 3 — Money & Compliance
- Full accounting depth
- GST filing support (GSTR-1/3B, e-invoicing/IRN, e-way bills — see
  [docs/gst-compliance.md](gst-compliance.md); invoice-level GST
  correctness itself ships in Phase 1, not here)
- Bank integration (payment reconciliation)
- Payment collection (UPI links, payment reminders)
- Vendor portal
- Purchase automation (Module 7 automatic POs)
- Tally connector GA ([docs/integrations.md](integrations.md))

## Phase 4 — Predictive Intelligence
- Sales forecasting
- Demand prediction (Module 6, full)
- Price suggestions (beyond diffing — proactive suggestions)
- Fraud/anomaly detection
- Business health score
- Profit optimization suggestions
- Embedding-based background catalog clustering at scale (Module 12's
  low-confidence sweep, upgraded from a periodic batch job to continuous
  reclustering as catalog sizes grow)
- Opt-in, anonymized cross-tenant alias/category dictionary to bootstrap
  a brand-new organization's catalog from patterns learned across other
  businesses in the same industry — see the future-idea note in
  [docs/product-catalog-lifecycle.md](product-catalog-lifecycle.md) and
  the privacy caveat in [docs/security.md](security.md); requires a
  dedicated privacy design pass before implementation, not just a data
  pipeline change

## Phase 5 — Autonomous Agent
- Owner speaks naturally; the AI Business Assistant performs complete
  multi-step business operations end-to-end (still gated by
  confirm-before-commit for anything financial — see
  [docs/ai-architecture.md](ai-architecture.md) and
  [PRODUCT_PRINCIPLES.md](../PRODUCT_PRINCIPLES.md)).
- Full connector ecosystem: Busy, Marg ERP, Zoho Books, QuickBooks, plus
  Tally, all GA.

## Backlog / not yet scheduled
- Additional regional languages beyond Hindi/Hinglish (moved to Phase 2
  above)
- Multi-warehouse advanced logistics
- White-label / reseller program
