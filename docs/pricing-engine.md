# Price Intelligence Engine

Covers Module 5 (Price Intelligence) from [PROJECT_SPEC.md](../PROJECT_SPEC.md).

## Problem

Businesses carry thousands of SKUs with prices that change frequently
(supplier cost changes, seasonal demand, customer-specific discounts).
Manually re-keying prices from a supplier PDF or WhatsApp message is slow
and error-prone.

## Design

1. **Ingestion** — a price update source (supplier PDF, Excel/CSV,
   WhatsApp message, website scrape, manual entry, or partner API) is
   submitted to `services/ai`.
2. **Extraction** — the AI pipeline (see
   [ai-architecture.md](ai-architecture.md)) extracts product name/SKU
   and new price per line.
3. **Matching** — each extracted line is matched against the
   organization's product catalog (`products`, `product_prices`) by
   name/SKU/barcode fuzzy match.
4. **Diffing** — for every match, compute old price, new price, delta,
   and resulting margin (using the organization's known cost price).
   Only products whose price actually changed are surfaced — unchanged
   lines are silently discarded to keep the review short.
5. **Customer-specific pricing** — if the organization has
   customer-specific price lists, the diff shows impact per price list,
   not just the base retail price.
6. **Owner review** — the diff is shown as a simple approve/reject list,
   grouped by category, with bulk-approve for high-confidence matches and
   individual review for low-confidence or unmatched lines.
7. **Commit** — approved changes write to `product_prices` and are logged
   append-only in `price_history` with source and approver.

## Guardrails

- A price change beyond a configurable threshold (e.g. >25% swing) is
  always flagged for individual review, never bulk-approved automatically.
- Unmatched lines (no existing product found) are routed into
  `candidate_products`, the same staging step AI billing and catalog
  growth use — see
  [product-catalog-lifecycle.md](product-catalog-lifecycle.md) — rather
  than being silently dropped or written straight into `products`. They
  appear alongside the price diff in one combined review, not a separate
  screen.
- Every commit is attributed to the ingestion source, so a bad supplier
  PDF can be traced and its changes reverted as a batch.

## Price-missing products

A product can exist in the catalog with `pricing_status = price_missing`
(see [database.md](database.md)) — created with a name but no known
price yet, for example from a handwritten product list uploaded before
any supplier bill has arrived (Scenario 3 in
[product-catalog-lifecycle.md](product-catalog-lifecycle.md)). This is
not an error state: the product is fully searchable and matchable, and
simply surfaces as a small dashboard nudge until a price arrives, either
from a future supplier bill (Scenario 4) or manual entry. Invoicing a
price-missing product prompts for the price inline, once, rather than
blocking the sale entirely.

## Undo a price batch

Every commit from one ingestion (one supplier PDF upload, one bulk
approval) shares a `batch_id` in `price_history` (see
[database.md](database.md)). "Undo last price batch" is a first-class,
one-click action available for a configurable window after the commit
(e.g. 24 hours) — not just the "traced and reverted" sentence this
document previously left it as. Reverting writes new offsetting
`price_history` entries tagged with the same `batch_id`; nothing is
deleted, so the full sequence — original prices, the bad batch, and the
revert — stays reconstructable. This matters specifically because a bad
OCR read (e.g. every price misread 2x from a skewed photo) is a plausible
everyday failure mode for this feature, not an edge case, and the cost of
not having a fast undo is real financial damage across potentially
hundreds of products at once.

## Bulk diff at scale

A large catalog (thousands of products) receiving a bulk price update
(e.g. a 100-line supplier bill) is presented as "N of M products
changed," with `Update All` for the common case and `Review one by one`
for when the owner wants to inspect individual changes — see Scenario 9
in [product-catalog-lifecycle.md](product-catalog-lifecycle.md) for a
worked example. This is the same diff/approval mechanism described above,
just described here at the scale it actually needs to perform at.
