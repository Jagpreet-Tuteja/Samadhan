# Decision Log

High-level log of accepted decisions. Full reasoning lives in
[docs/adr](adr).

- Monorepo structure (`apps/*`, `services/*`, `packages/*`, `docs/*`) is
  accepted — [ADR 0001](adr/0001-monorepo.md).
- PostgreSQL as the primary datastore is accepted —
  [ADR 0002](adr/0002-database.md).
- Phone + OTP as the primary authentication method, with JWT
  access/refresh tokens, is accepted — [ADR 0003](adr/0003-authentication.md).
- Role-based access control scoped per organization is accepted —
  [ADR 0004](adr/0004-permissions.md).
- The AI layer (`services/ai`) is a separate service from the core
  business backend (`services/core`) is accepted —
  [ADR 0005](adr/0005-ai-integration-architecture.md).
- Tally integration runs via a local agent talking to Tally's XML
  interface, syncing to Nexora's cloud API, rather than a direct
  cloud-to-Tally connection — [ADR 0006](adr/0006-tally-integration.md).
- Single multi-tenant deployment with `organization_id` row scoping,
  rather than a database-per-tenant model, is accepted —
  [ADR 0007](adr/0007-multi-tenancy.md).
- AI extraction never writes directly into the canonical `products`
  table; every product mention is staged in `candidate_products` and
  resolved through tiered-confidence identity matching (fuzzy text +
  attributes + price consistency + context + embeddings) before
  becoming, or updating, a real product, with even near-certain matches
  surfaced as an undoable confirmation rather than fully silent —
  [ADR 0009](adr/0009-product-catalog-identity-resolution.md).
- V1 is a deliberate cut-down of the target architecture (one backend
  service, Postgres only, web-only, no Tally agent, no Kubernetes), not
  the target architecture itself — [ADR 0010](adr/0010-mvp-scope-cut.md).
- V1 requires connectivity to bill, as an explicit, measured tradeoff;
  the API is designed sync-safe from day one so true offline-first can
  ship with the mobile app in Phase 2 without a rearchitecture —
  [ADR 0011](adr/0011-offline-first-strategy.md).
- Nexora stays an unambiguous Tally companion through at least Phase 2;
  independent accounting depth in Phase 3+ is built for its own value,
  not framed as replacing Tally, until a business's own usage earns that
  outcome — [ADR 0012](adr/0012-tally-adoption-sequencing.md).
- Postgres Row-Level Security enforces tenant isolation at the database
  level as a backstop behind application-layer `organization_id`
  scoping — [ADR 0013](adr/0013-row-level-security.md).
- V1's combined backend service is Java (Spring Boot) only, not Python;
  AI extraction calls a hosted LLM API directly rather than running a
  local pipeline — [ADR 0014](adr/0014-v1-java-only-backend.md).
- The `organization_members` RLS policy (ADR 0013) is widened with a
  second, read-only allowance: a user can always see their own
  membership rows by `user_id`, regardless of org context — the only way
  login can discover which organization(s) a phone number's user
  belongs to before any org is selected. Write access (`WITH CHECK`)
  stays org-context-only.
- Setting a product's retail price is gated by
  `billing.approve_price_change`, not `inventory.write`, and lives in its
  own `pricing` package: a manual price is a price approval, and
  docs/permissions.md gives Accountant and Staff none, even where they can
  read the catalog. Every change appends to `price_history`.
- Stock on hand is always `SUM(stock_ledger.quantity_delta)`, never a
  stored counter, and is allowed to go negative: blocking a real sale
  because stock was never formally entered pushes the owner off the
  system without making the count right. Low stock is surfaced instead.
- Product SKUs are auto-generated per organization and prefix from an
  atomic upsert counter, and are immutable once assigned (they're
  printed on barcodes and used by integrations).
- GST tax is computed per invoice line, with intrastate tax as two
  half-rate amounts each rounded half-up (so CGST always equals SGST),
  and the invoice totals are exact sums of the lines, rounded to the
  nearest rupee with an explicit round-off. Tax-inclusive prices are
  back-calculated so the customer pays exactly the quoted price. See
  `GstCalculator` and its hand-worked fixture suite.
- Invoicing is refused until the owner explicitly sets the
  organization's GST profile, so a business that hasn't entered its
  GSTIN can never silently issue tax-less bills. Composition and
  unregistered sellers issue bills of supply; composition sellers can't
  sell interstate.
- Invoices are immutable snapshots (seller, buyer and product GST details
  copied at issue time), identified by client-generated UUIDs for
  idempotent retries, numbered server-side per financial year.
- Overriding a catalog price on an invoice needs
  `billing.approve_price_change`; Staff bill at catalog prices only.
