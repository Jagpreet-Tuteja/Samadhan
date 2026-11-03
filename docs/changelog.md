# Changelog

Meaningful project changes, newest first.

## 2026-11-03
- Module 3 (Billing) V1 scope: GST tax invoices (CGST+SGST or IGST by
  place of supply) and bills of supply (composition/unregistered), an
  owner-set GST profile with checksum-validated GSTINs, idempotent
  invoice creation on client UUIDs, per-financial-year invoice numbers,
  and stock + customer-ledger effects in one transaction.
  `V4__billing.sql`; see `docs/api.md` and `GstCalculatorTest`'s
  hand-worked GST fixtures. Customers gained a billing state and
  `payment` ledger entries.
- Integration tests run on Docker Engine 29 (Testcontainers 1.21.4) and
  share one Postgres container across the run.
- Module 2 (Inventory) V1 scope, plus manual retail pricing:
  `V3__catalog_and_inventory.sql` adds categories, products (auto-SKU,
  HSN code, GST rate, `pricing_status`), `product_prices`/append-only
  `price_history`, a single lazily-created default warehouse, and an
  append-only `stock_ledger`, all RLS'd. Endpoints under
  `/api/v1/product-categories` and `/api/v1/products` (stock movements,
  price, price history). This is the product/price/stock base that both
  Module 3 (Billing) and Module 12 (Catalog Intelligence) build on. See
  `docs/api.md` "Implemented endpoints" and `InventoryApiIT`.

## 2026-09-13
- Module 1 (Customer Management) shipped: profiles, phone/name search,
  and an append-only ledger with a computed balance, gated by
  `customers.read`/`customers.write`. First product module built on the
  foundation + auth slices from the same day. See `docs/api.md`
  "Implemented endpoints" and `CustomerApiIT`.
- Real phone+OTP authentication (ADR 0003): OTP request/verify, signup,
  multi-org selection/switching, JWT access tokens, rotated refresh
  tokens with per-device revocation. `V2__auth_and_customers.sql` adds
  the auth tables and widens `organization_members`'s RLS policy with a
  read-only self-row allowance. `TenantContext` is now populated from
  real JWTs, not a test harness. See `docs/api.md` "Implemented
  endpoints" and `services/core/TEST_VERIFICATION.md` for two Spring
  Security 6 bugs found and fixed along the way.
- First application code: `services/core` (Spring Boot 3, Java 21) and
  the first Flyway migration
  (`packages/database/migrations/V1__tenancy_and_identity.sql`) —
  `organizations`/`users`/`roles`/`permissions`/`organization_members`,
  seeded roles and permissions per `docs/permissions.md`, and Row-Level
  Security per ADR 0013, proven by an automated cross-tenant isolation
  test rather than asserted by the SQL alone. Decided and documented that
  V1's combined backend service (`docs/mvp-scope.md`) is Java only, not
  Python — see `docs/tech-stack.md` and `docs/context.md`.

## 2026-07-19
- CTO-level critique pass, applied as 13 commits: added
  `docs/mvp-scope.md` (hard V1 boundary + validation gate cutting the
  target architecture down to one service/Postgres-only/web-only/no
  Tally agent/no Kubernetes), `docs/gst-compliance.md` (GST correctness
  moved to Phase 1, filing stays Phase 3), `docs/offline-strategy.md`
  (explicit V1 connectivity tradeoff, sync-safe API design),
  `docs/ui-ux-guidelines.md` (design principles matched to the real
  buyer instead of a Stripe/Notion aesthetic), `docs/future-modules.md`
  (long-term Business-OS vision separated from the active spec, with the
  Zoho sequencing lesson as rationale). Resolved the Tally
  companion-vs-replacement contradiction (ADR 0012). Fixed the
  near-certain-auto-merge silent-write contradiction and added price as
  a fifth identity-resolution signal. Added batch undo for price
  changes and Postgres Row-Level Security as a tenant-isolation backstop
  (ADR 0013). Added an AI unit-economics cost model and guardrails to
  `docs/monetization.md`. Added a Phase 0 validation gate to the roadmap
  and moved Hindi/Hinglish input from an unscheduled backlog item into
  Phase 2. New ADRs 0010-0013.
- Added Module 12 — Catalog Intelligence: the self-growing product
  catalog and product identity resolution design. New
  `docs/product-catalog-lifecycle.md` and
  `docs/adr/0009-product-catalog-identity-resolution.md`; updated
  `docs/database.md` (`candidate_products`, `product_sku_sequences`,
  `confirmed_distinct_pairs`, `pricing_status`/`aliases`/`attributes` on
  `products`), `docs/ai-architecture.md` (identity resolution pipeline
  stage), `docs/pricing-engine.md` (candidate routing, price-missing,
  bulk diff), `PROJECT_SPEC.md`, `docs/features.md`, `docs/roadmap.md`,
  `docs/glossary.md`, `MASTER_INDEX.md`, `docs/security.md`
  (cross-tenant-dictionary privacy caveat), and
  `docs/testing-strategy.md` (catalog quality metrics).
- Initial repository: full product specification, architecture, and
  planning documentation created (root docs + `docs/*` + `docs/adr/*`).
  No application code yet.
