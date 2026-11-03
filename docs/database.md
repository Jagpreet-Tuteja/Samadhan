# Database

Status: partially implemented — see "Migration history" below for what
exists. The rest of this document is the canonical schema plan;
implementation should follow it and update it in
the same change when the real schema is created.

## Principles

- Every business-owned table carries `organization_id` (tenant scoping).
- Money is stored as integer minor units (paise), never floating point.
- Every mutable financial record (invoice, price, stock movement) is
  append-only at the ledger level — corrections create new entries
  (credit/debit notes, stock adjustments), not silent edits, so history
  and audit trails are always reconstructable.
- Soft-delete (`deleted_at`) on customer/supplier/product records;
  hard-delete is never exposed to the AI layer.

## Core tables (planned)

### Tenancy & identity
- `organizations` — one row per business (name, industry type, GST
  number, address, subscription tier).
- `users` — login identity, belongs to one or more organizations.
- `organization_members` — user ↔ organization with `role_id`.
- `roles`, `permissions`, `role_permissions` — see
  [permissions.md](permissions.md).

### Customers & suppliers
- `customers` (org-scoped): name, phone, address, GST number, credit
  limit, notes.
- `customer_ledger_entries`: running balance entries per customer
  (invoice, payment, credit note).
- `suppliers` (org-scoped): name, phone, address, GST number, rating.
- `supplier_ledger_entries`: running balance per supplier (purchase bill,
  payment made).

### Catalog & inventory
- `product_categories`, `brands`
- `products` (org-scoped): name, SKU, barcode/QR, base unit, tax rate,
  images, min-stock threshold, `pricing_status`
  (`priced` / `price_missing` / `pending_review`), `aliases` (text array —
  every name this product has ever been called, across every input
  source), `attributes` (JSON — extracted size/color/brand/material/pack
  size, used for matching and disambiguation).
- `product_units` — configurable unit conversions per product (e.g. 1 box
  = 12 piece).
- `candidate_products` (org-scoped) — staging table for every product
  mention detected by AI extraction before it becomes, or is matched to,
  a real `products` row: raw extracted name, inferred attributes, source
  (`ai_extraction_jobs` reference), match candidates with scores,
  status (`pending` / `approved_new` / `merged` / `rejected`). See
  [product-catalog-lifecycle.md](product-catalog-lifecycle.md) and
  [adr/0009-product-catalog-identity-resolution.md](adr/0009-product-catalog-identity-resolution.md).
  AI extraction writes only to this table, never directly to `products`.
- `product_sku_sequences` (org-scoped) — one counter per category prefix
  (e.g. `PVC`, `GLU`), used to auto-generate collision-free SKUs
  (`PVC-0001`, `PVC-0002`, ...) when a business has no SKU system of its
  own.
- `confirmed_distinct_pairs` (org-scoped) — permanent record of product
  pairs the owner has explicitly said are *not* the same product, keyed
  as an unordered pair of product/candidate identifiers, so identity
  resolution never re-suggests a merge the owner already rejected.
- `warehouses` (org-scoped)
- `stock_ledger` — append-only stock movement (in/out, reason, reference
  invoice/purchase/adjustment, warehouse, batch, expiry).
- `product_prices` — current price per product per price-list
  (retail/wholesale/customer-specific).
- `price_history` — append-only log of every price change, its source
  (manual, supplier PDF, WhatsApp, CSV, API), `batch_id` (groups every
  change committed together from one ingestion, e.g. one supplier PDF
  upload), and who approved it. `batch_id` is what makes "undo this
  batch" (see [pricing-engine.md](pricing-engine.md)) a query instead of
  a manual reconstruction — reverting a batch writes new offsetting
  entries tagged with the same `batch_id`, it never deletes history.

### Billing
- `invoices`, `invoice_items`
- `quotations`, `quotation_items`
- `purchase_orders`, `purchase_order_items`
- `purchase_bills`, `purchase_bill_items`
- `credit_notes`, `debit_notes`
- `delivery_challans`, `delivery_challan_items`
- `payments` — polymorphic, links to invoice, purchase bill, or ledger
  entry; method (cash/UPI/card/bank/credit).

### Expenses
- `expense_categories`
- `expenses` (org-scoped): category, amount, date, note, attachment.

### AI & integrations
- `ai_conversations` — chat/voice session log per user.
- `ai_extraction_jobs` — OCR/vision/voice job, input reference, output
  draft, confidence score, status (pending/confirmed/rejected).
- `connector_configs` — per-organization connector settings (Tally
  instance URL, sync direction, field mappings).
- `connector_sync_log` — append-only log of every record pushed/pulled
  to/from an external system, for reconciliation and debugging.

### Platform
- `audit_logs` — who did what, when, on which record.
- `notifications`

## Indexing & constraints

- Composite indexes on `(organization_id, created_at)` for every
  high-volume table (invoices, stock_ledger, ai_extraction_jobs).
- Unique constraint on `(organization_id, phone)` for customers where
  phone is present.
- Unique constraint on `(organization_id, category_prefix)` for
  `product_sku_sequences`; SKU generation increments this counter inside
  the same transaction that creates the product, to stay collision-free
  under concurrent writes.
- Unique constraint on `(organization_id, candidate_id_a, candidate_id_b)`
  for `confirmed_distinct_pairs`, with `candidate_id_a < candidate_id_b`
  enforced at the application layer so a pair is only ever stored once
  regardless of comparison order.
- A trigram or full-text index on `products.name` and `products.aliases`
  (org-scoped) backs the live fuzzy-match step in identity resolution; the
  embedding-based background clustering pass (see
  [ai-architecture.md](ai-architecture.md)) reads from a vector index
  maintained separately, not this one.
- Foreign keys enforced at the database level; the AI layer never writes
  directly — it only produces drafts (and, for products specifically,
  `candidate_products` rows) that `services/core` validates and commits.
- **Row-Level Security (RLS)** policies enforce `organization_id`
  scoping at the database level on every tenant-owned table, as a second,
  independent layer behind the application-layer scoping in
  `services/core` — see
  [adr/0013-row-level-security.md](adr/0013-row-level-security.md). A
  missed `WHERE organization_id = ?` in application code becomes a query
  returning zero rows, not a cross-tenant data leak.

## Migration history

- `V1__tenancy_and_identity.sql` (2026-09-13): `organizations`, `users`,
  `roles`, `permissions`, `role_permissions` (seeded with the four default
  roles and their permission mappings per
  [permissions.md](permissions.md)), and `organization_members` — the
  first actual tenant-scoped table, with Row-Level Security enabled per
  [adr/0013-row-level-security.md](adr/0013-row-level-security.md). Also
  creates the restricted `nexora_app` application role RLS depends on
  (see [tech-stack.md](tech-stack.md)) and enables `pg_trgm`. Managed by
  Flyway; see [adr/0002-database.md](adr/0002-database.md) for the
  technology choice. Migrations live in
  `packages/database/migrations/`, applied by `services/core` via a
  separate, elevated Flyway connection distinct from the application's
  own restricted runtime connection.
- `V2__auth_and_customers.sql` (2026-09-13): `otp_challenges` and
  `refresh_tokens` (auth infrastructure, not tenant-owned, no RLS —
  looked up by phone/token hash, never listed per-org); `customers` and
  `customer_ledger_entries` (Module 1, RLS'd like `organization_members`).
  Also widens the `organization_members` RLS policy with a second,
  narrow `USING` allowance — a user can always see their own membership
  rows by `user_id`, regardless of org context — solving login's
  chicken-and-egg problem (discovering which org(s) a user belongs to
  requires reading this table before any org is selected). `WITH CHECK`
  is unchanged (org-context only), so this only affects reads. See
  `com.nexora.core.tenancy.TenantContext`'s `currentUserId` field and
  [ADR 0013](adr/0013-row-level-security.md).
- `V3__catalog_and_inventory.sql` (2026-11-03): Module 2's V1 scope plus
  the manual-price half of the pricing tables. `product_categories`
  (with `sku_prefix` and the unit/HSN/GST-rate defaults a new product
  inherits), `product_sku_sequences` (one counter per org + prefix,
  advanced by a single atomic `INSERT ... ON CONFLICT DO UPDATE ...
  RETURNING` inside the product-creating transaction), `products`
  (`sku` unique per org, `barcode` unique per org when present,
  `hsn_code`, `gst_rate_basis_points` — 1800 = 18%, integer like money —
  `min_stock_quantity`, `pricing_status`, soft-delete column, trigram
  index on `name`), `product_prices` (current price per product per
  price list — `retail` only for now), `price_history` (append-only, with
  `batch_id`; a manual edit is a batch of one), `warehouses` (one default
  per org, enforced by a partial unique index, created lazily on the
  first stock movement), and `stock_ledger` (append-only, signed
  `NUMERIC(14,3)` `quantity_delta` so kg/meter stock works; on-hand is
  always `SUM(quantity_delta)`, never a stored counter). All RLS'd like
  `customers`. Deliberately deferred, each to the slice that will use
  it: `products.aliases`/`attributes`, `candidate_products`,
  `confirmed_distinct_pairs` (Module 12); `product_units`, `brands`,
  batch/expiry (outside V1 inventory scope); wholesale/customer price
  lists (extend the `price_list` CHECK).
- `V4__billing.sql` (2026-11-03): Module 3. `organizations.tax_scheme`
  (`regular`/`composition`/`unregistered`, NULL until set — invoicing is
  refused until then) and `organizations.state_code`;
  `customers.state_code`; `customer_ledger_entries.reference_type`/
  `reference_id`; a new `org.manage_settings` permission seeded to Owner.
  `invoice_number_sequences` (per org per financial year, same atomic
  upsert as SKUs), `invoices` (client-generated UUID primary key; seller
  and buyer GST details snapshotted; `invoice_type` `tax_invoice` or
  `bill_of_supply`; CHECKs that a bill of supply carries no tax and that
  the total equals taxable + taxes + round-off) and `invoice_items`
  (product name/HSN/unit/rate snapshotted; CHECKs that CGST = SGST and
  the line total adds up). All RLS'd. Not yet: quotations, purchase
  documents, credit/debit notes, challans, e-invoicing IRN columns.
