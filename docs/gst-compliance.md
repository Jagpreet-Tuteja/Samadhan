# GST Compliance

Implementation status (2026-11-03): the V1 invoice-level logic below is
built — intrastate/interstate split, HSN snapshot per line, regular vs.
composition vs. unregistered (bill of supply), explicit place of supply,
and consistent rounding — in `com.nexora.core.gst` and
`com.nexora.core.billing`, with the fixture suite in `GstCalculatorTest`.
RCM applies to purchase bills, which don't exist yet. The printed
invoice format (legally required fields, the composition declaration)
arrives with the invoice view in `apps/web`.

## Why this is a V1 concern, not a Phase 3 one

[docs/roadmap.md](roadmap.md) originally placed "GST filing support" in
Phase 3, which is correct — but Module 3 (Billing) ships in
[mvp-scope.md](mvp-scope.md)'s V1, and every GST invoice it produces has
to be *correct* from the first one. There is a real difference between
two things that got conflated earlier in this project's planning:

- **GST correctness on invoices** (tax split, HSN codes, place of
  supply) — this must work in V1. Get it wrong once and the business's
  accountant rejects the invoice, and that business does not come back.
- **GST filing/compliance workflows** (GSTR-1/3B generation, e-invoicing
  IRN, e-way bills) — these are genuinely Phase 3 work, not needed for a
  business to safely use Nexora day to day, since their accountant (or
  Tally, via [integrations.md](integrations.md)) still handles filing
  until then.

## Core invoice-level logic (V1)

- **Intrastate vs. interstate**: compare the organization's registered
  state (from its GSTIN) to the customer's billing state. Same state →
  CGST + SGST, split evenly. Different state → IGST, full rate. This
  determination must be automatic — never something the owner types in
  per invoice.
- **HSN/SAC codes**: every product needs an `hsn_code` (goods) or
  `sac_code` (services) field — see [database.md](database.md). Required
  on invoices above applicable turnover thresholds; store it from V1
  regardless of whether a given pilot business is currently above that
  threshold, so nothing has to be retrofitted later.
- **Tax scheme per organization**: `organizations.tax_scheme` —
  **Regular** (standard GST invoice with full tax breakup) or
  **Composition** (fixed low-rate scheme; composition dealers are legally
  prohibited from showing a GST breakup or collecting GST from
  customers — the invoice format itself is different, not just the
  number). Getting this wrong is a compliance violation for the business,
  not just a cosmetic bug.
- **Reverse charge mechanism (RCM)**: certain purchase categories require
  the buyer, not the seller, to pay GST directly. Flag applicable
  purchase bill line items rather than assuming standard forward-charge
  always applies.
- **Place of supply**: drives intrastate/interstate determination above;
  store explicitly on the invoice rather than re-deriving it later from
  customer address, since the rule (billing address vs. delivery address
  vs. registered address, depending on transaction type) has real edge
  cases worth getting right once, explicitly, rather than inferring
  implicitly every time.
- **Rounding**: GST amounts round per standard invoice-level rounding
  rules, applied consistently — this is a common source of
  paisa-level invoice mismatches that erode trust disproportionately to
  how small the actual error is.

## Deferred to Phase 3 (filing and statutory workflows)

- **E-invoicing (IRN)**: mandatory above a turnover threshold — requires
  real-time invoice registration with the government portal and a
  returned IRN/QR code embedded on the invoice. Design the invoice number
  scheme and invoice data model now (V1) so this can be added without a
  breaking schema change later, even though the actual integration is
  Phase 3.
- **E-way bills**: required for goods movement above a value threshold.
- **GSTR-1 / GSTR-3B generation**: the actual monthly/quarterly return
  filing workflow — this is what would let a business eventually reduce
  its dependence on a separate accountant/Tally workflow, per
  [adr/0012-tally-adoption-sequencing.md](adr/0012-tally-adoption-sequencing.md).

## Testing requirement

GST correctness needs its own fixture-based test suite — real invoice
scenarios (intrastate, interstate, composition scheme, RCM) checked
against known-correct tax breakdowns, run in CI, separate from the AI
extraction-accuracy fixtures described in
[testing-strategy.md](testing-strategy.md). This is deterministic
business logic, not an AI accuracy problem, and should be tested to a
100%-correct bar, not a "less than 1% error" bar.

## Non-goal

Nexora does not aim to become a certified GST Suvidha Provider (GSP) or
replace a business's statutory compliance relationship with their
accountant in V1 or Phase 1/2. Correctness of what Nexora itself
generates is the V1 bar; full filing is a deliberate, later expansion —
see [future-modules.md](future-modules.md) for how this fits the
long-term "own the General Ledger" ambition.
