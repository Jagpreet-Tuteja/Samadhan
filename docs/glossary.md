# Glossary

- **Organization** — a single business/tenant using Nexora.
- **Owner** — the top-level role for an organization; see
  [permissions.md](permissions.md).
- **Draft** — an AI-produced structured proposal (invoice, price update,
  customer record) that has not yet been confirmed/committed.
- **Confidence score** — a per-field score the AI pipeline attaches to
  extracted data, used to decide whether a field needs manual review.
- **Connector** — an adapter in `services/connectors` that syncs data
  between Nexora and an external system (Tally, Busy, Marg, Zoho Books,
  QuickBooks, WhatsApp).
- **Price list** — a named set of prices for a product (retail, wholesale,
  customer-specific).
- **Ledger entry** — an append-only record of a balance-affecting event
  for a customer or supplier.
- **Stock ledger** — an append-only record of every inventory movement.
- **ADR** — Architecture Decision Record; see [docs/adr](adr).
- **Candidate product** — a staged, unconfirmed product mention detected
  by AI extraction, held in `candidate_products` until it is matched to
  an existing product or approved as new. See
  [product-catalog-lifecycle.md](product-catalog-lifecycle.md).
- **Identity resolution** — the process of deciding whether a newly
  extracted product mention is an existing product, an alias of one, or
  genuinely new, using fuzzy text, structured attributes, context, and
  embeddings.
- **Alias** — an alternate name a canonical product has been called
  under (across suppliers, phrasing, or time), retained rather than
  discarded so future matches against that name resolve instantly.
- **Product Approval Queue** — the review surface where AI-detected new
  products, merge suggestions, and auto-linked aliases are surfaced for
  the owner to confirm, in bulk or individually.
- **Price-missing** — a `pricing_status` a product can hold when it has
  been created (e.g. from a handwritten list) but no price is known yet;
  not an error state, just an incomplete one.
- **Confirmed distinct pair** — a permanent record that the owner has
  explicitly said two products are not the same, preventing that pair
  from ever being re-suggested as a merge.
- **MVP validation gate** — the five conditions in
  [mvp-scope.md](../mvp-scope.md) that must all be true before Phase 2
  work begins, regardless of whether Phase 1's feature list is
  technically finished.
- **Batch (price batch)** — every price change committed together from
  one ingestion shares a `batch_id`, making "undo this batch" a single
  reversible action rather than a manual reconstruction.
- **RLS (Row-Level Security)** — a database-level policy that enforces
  `organization_id` tenant scoping as a backstop behind application-layer
  scoping, so a missed filter in code degrades to an empty result instead
  of a cross-tenant data leak.
- **HSN / SAC code** — the government classification code required on
  GST invoices for goods (HSN) or services (SAC); see
  [gst-compliance.md](../gst-compliance.md).
- **Composition scheme** — a GST registration type for small businesses
  that pay a fixed low rate and are legally prohibited from showing a
  GST breakup on invoices, unlike the standard Regular scheme.
