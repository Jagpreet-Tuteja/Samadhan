# Progress Log

Latest session summary at the top. Append new sessions below; do not
delete history.

## 2026-11-03 — Module 3: Billing (V1 scope, GST-correct)

- `V4__billing.sql`: organization GST profile columns, `customers.state_code`,
  ledger references, `org.manage_settings` permission (Owner),
  `invoice_number_sequences`, `invoices`, `invoice_items` — see
  docs/database.md. DB CHECKs back the invariants the code relies on
  (bill of supply has no tax, CGST = SGST, totals add up).
- `com.nexora.core.gst`: `Gstin` (format + mod-36 checksum + state code;
  the checksum algorithm was confirmed against two published sample
  GSTINs before being written) and `GstCalculator` (pure, per-line,
  half-up to the paisa, CGST/SGST as independently-rounded halves,
  tax-inclusive back-calculation, rupee round-off).
- `com.nexora.core.billing`: `TaxTreatment` (pure: invoice type, supply
  type and place of supply from the seller profile, request and
  customer; composition can't go interstate), `InvoiceService`
  (idempotent on the client UUID; number reserved last so validation
  failures don't burn numbers; stock + customer-ledger effects in the
  same transaction), `InvoiceController`. `com.nexora.core.organization`:
  the tax-profile endpoint.
- Module 1 touched: customer `stateCode` (derived from GSTIN), GSTIN
  validation, `payment` ledger entries (negative), ledger
  `referenceType`/`referenceId`. Module 2 touched: `StockService.recordSale`.
- Decisions recorded in docs/decisions.md: per-line rounding with sum
  totals, refuse-until-profile-set, immutable snapshots, Staff bill at
  catalog prices only.
- Tests: `GstCalculatorTest` (11 hand-worked fixture cases: intra/inter,
  half-paisa boundaries, inclusive back-calculation, fractional
  quantities, 0.25%/3%/28% slabs, bill of supply, 0%, round-off),
  `GstinTest`, `TaxTreatmentTest`, `InvoiceNumbersTest`, and
  `BillingApiIT` (11 HTTP scenarios incl. idempotent retry, cross-tenant
  id reuse → 409, composition/unregistered, price/rate preconditions,
  Staff price override → 403). **All 68 tests pass** against real
  Testcontainers Postgres.
- Deliberately deferred: invoice PDF/print, quotations, purchase bills,
  credit/debit notes (also the only legal way to correct an issued
  invoice), challans, e-invoicing IRN, RCM (purchase-side), cursor
  pagination for the invoice list.
- **Next session should start with**: docs/context.md "Immediate next
  step" — `apps/web` with the billing screen and a print-ready invoice.

## 2026-11-03 — Module 2: Inventory (V1 scope) + manual pricing

- Why this slice first: the documented next step was Billing + Module
  12 together, but both need a `products` table, prices, and a stock
  ledger that didn't exist. This builds that shared base so Billing can
  be built on it next.
- `V3__catalog_and_inventory.sql`: `product_categories`,
  `product_sku_sequences`, `products`, `product_prices`, `price_history`,
  `warehouses`, `stock_ledger` — all RLS'd like `customers` (see
  docs/database.md "Migration history" for the column-level detail and
  what was deliberately deferred to Module 12).
- `com.nexora.core.inventory`: categories (SKU prefix derived from the
  name when omitted), products (unit/HSN/GST rate inherited from the
  category; auto-SKU via `SkuGenerator`'s atomic upsert, skipping values a
  hand-entered SKU already took), search by name/SKU/barcode plus
  `categoryId`/`pricingStatus`/`lowStock` filters, stock movements
  (`opening_stock` only as the first movement; `adjustment` signed and
  non-zero). List responses batch-load stock levels and prices: two
  queries per page, not two per product.
- `com.nexora.core.pricing`: `PUT /products/{id}/price` (gated by
  `billing.approve_price_change`, not `inventory.write`) upserts the
  retail price, appends `price_history` (source `manual`, a batch of
  one), and flips the product to `priced`. Price changes and the
  opening-stock check take a `PESSIMISTIC_WRITE` row lock on the product,
  so concurrent writes can't record the wrong old price or two opening
  balances.
- Design calls worth knowing: SKU is immutable after creation; negative
  on-hand is allowed (refusing a real sale doesn't fix the count —
  `lowStock` surfaces it instead); GST rate is range-validated only, not
  checked against a hard-coded slab list, since slabs change by
  notification.
- Tests: `SkuPrefixesTest` (pure unit, passes) and `InventoryApiIT` (real
  HTTP: inherited defaults + sequential SKUs, name-derived prefixes,
  hand-entered SKU collision skipping, unit/duplicate validation, stock
  accumulation + low stock, opening-stock rules, price history,
  search, the role matrix including Accountant/Staff blocked from
  pricing, and cross-tenant isolation including per-org SKU counters).
  All 33 tests across every class pass together against real
  Testcontainers Postgres (`./mvnw test '-Dtest=*Test,*IT'`), after two
  test-infrastructure fixes found while getting them to run:
  1. Testcontainers 1.20.1 → 1.21.4. Docker Engine 29 rejects the old
     default Docker API version 1.20.x requests, answering with an
     all-empty `/info` stub that surfaces as "Could not find a valid
     Docker environment". This is very likely what the 2026-09-13
     `TEST_VERIFICATION.md` entry attributed to a sandbox restriction.
  2. `AbstractIntegrationTest` now uses one shared container for the
     whole run instead of a per-class `@Container`: Spring's cached
     application context outlived each class's container, so the second
     class with an identical context (`InventoryApiIT` after
     `CustomerApiIT`) hit a dead port and timed out. The earlier "all IT
     classes pass together" result was against `docker compose` Postgres,
     which hid this.
- Found while verifying: a plain `./mvnw test` doesn't run any `*IT`
  class (Surefire's default includes), and the repo has no CI — see
  docs/context.md.
- **Next session should start with**: docs/context.md "Immediate next
  step" — Module 3 (Billing) on top of this base.

## 2026-09-13 — Module 1: Customer Management

- Built on the `customers`/`customer_ledger_entries` schema
  (`V2__auth_and_customers.sql`, landed with the auth slice earlier the
  same day): `Customer` (`@SQLRestriction("deleted_at IS NULL")` so the
  soft-delete convention in docs/database.md applies to every query
  through the entity, including `findById`, even though deletion itself
  isn't built in this slice) and `CustomerLedgerEntry`.
- Endpoints (`com.nexora.core.customers.CustomerController`, gated by
  `customers.read`/`customers.write` via `@PreAuthorize`): create,
  get, partial update (PATCH -- a field left out of the body is
  unchanged, no support yet for explicitly clearing one back to blank),
  search (`?search=` matches name or phone, plain page/size pagination),
  and ledger entries (create restricted to `opening_balance`/`adjustment`
  in this slice -- `invoice`/`payment`/`credit_note` are Module 3's job
  on the same table later; list; the customer response's
  `balanceMinorUnits` is a live `SUM` over the ledger).
- Every repository method takes `organizationId` explicitly and filters
  on it -- defense-in-depth alongside Row-Level Security (ADR 0007, ADR
  0013), matching the pattern already established in
  `OrganizationMemberRepository`.
- Added `PageResponse<T>`, a small reusable pagination wrapper for the
  `{"data": ...}` envelope -- the first paginated list endpoint in the
  project, reusable by every future one.
- `CustomerApiIT` (real HTTP): create/find/update, duplicate-phone
  rejection, name/phone search, ledger accumulation into the balance,
  invalid ledger `entryType` rejection, Staff blocked from
  `customers.write` but not `customers.read` (a real permission-matrix
  case from docs/permissions.md), and cross-tenant isolation (a second
  organization can neither fetch nor search into the first one's
  customer). Mints tokens directly via `JwtService` for test setup
  rather than re-driving the OTP flow, which `AuthFlowIT` already covers.
  Verified for real against `docker compose` Postgres in this sandboxed
  session (see `TEST_VERIFICATION.md`), then `AbstractIntegrationTest`
  reverted to the checked-in Testcontainers version. All 17 tests across
  the four IT classes pass together.
- **Next session should start with**: `docs/context.md` "Immediate next
  step" — Module 3 (Billing) and Module 12's core loop (Catalog
  Intelligence) together, since an invoice needs products and products
  need Module 12's catalog-staging loop.

## 2026-09-13 — Real authentication (phone+OTP)

- Built the full phone+OTP flow (adr/0003-authentication.md):
  `V2__auth_and_customers.sql` adds `otp_challenges`, `refresh_tokens`,
  and widens `organization_members`'s RLS policy with a second, read-only
  `USING` allowance keyed on `user_id` — solving login's chicken-and-egg
  problem (discovering a user's org memberships requires reading that
  table before any org context exists). `WITH CHECK` stays
  org-context-only, so writes are unaffected.
- `TenantContext` gained a second field, `currentUserId`;
  `TenantAwareDataSource` stamps both `app.current_org_id` and
  `app.current_user_id` at connection checkout. Added
  `TenantSessionStamper` for the one case that's genuinely different:
  signup creates the organization and its owner's membership row in one
  atomic transaction, so the org/user ids don't exist until partway
  through it — after the connection was already checked out with no
  context. `TenantSessionStamper` re-applies the session variables
  mid-transaction, on the connection that transaction is already using,
  via Spring's shared (transaction-scoped) `EntityManager`.
- Endpoints (`com.nexora.core.auth.AuthController`): `otp/request`
  (rate-limited, 5/15min), `otp/verify` (branches to `authenticated` /
  `signup_required` / `select_organization_required`), `signup`,
  `select-organization`, `switch-organization` (move between
  memberships without a fresh OTP — an accountant managing several shops
  is ADR 0003's own example), `refresh` (rotates; the replaced token is
  rejected on reuse), `logout` (revokes).
- Added the shared pieces every future module reuses: `GlobalExceptionHandler`
  (the `{data|error}` envelope from docs/api.md),
  `SecurityConfig`/`JwtAuthenticationFilter` (stateless bearer-token
  auth), `PermissionService` (loads a role's permission codes as Spring
  Security authorities, so `@PreAuthorize("hasAuthority(...)")` on future
  controllers is enforced server-side per docs/permissions.md).
- OTP delivery: `OtpSender` interface, one implementation
  (`LoggingOtpSender`) — there's no real pilot business to text yet, so
  a real SMS/WhatsApp provider is explicitly deferred to Phase 0
  onboarding rather than built speculatively now.
- Hit and fixed two genuine Spring Security 6 bugs while building this
  (both documented in detail in `services/core/TEST_VERIFICATION.md` so
  a future session doesn't rediscover them the hard way):
  1. `JwtAuthenticationFilter` must never be a `@Component`/`@Bean` --
     Spring Boot auto-registers every `Filter`-typed bean as an
     independent servlet filter, running it twice and producing
     confusing, inconsistent authentication results. Fixed by
     constructing it directly inside `SecurityConfig` instead.
  2. `SecurityContextHolder.getContext().setAuthentication(...)` doesn't
     reliably stick in Spring Security 6 -- `SecurityContextHolderFilter`'s
     deferred-supplier mechanism can hand a later caller (`AuthorizationFilter`)
     a freshly-created empty context instead. Every authenticated request
     was being rejected with "Full authentication is required" even
     though the filter's own logging showed authentication being set
     correctly moments earlier. Fixed by building a whole context and
     installing it via `SecurityContextHolder.setContext(...)`, the
     pattern Spring's own docs now recommend.
  A third, unrelated fix: `TestRestTemplate`'s default JDK
  `HttpURLConnection` client can't handle a 401/403 response to a
  streamed POST body -- added `httpclient5` as a test dependency, which
  Spring Boot auto-prefers once present.
- `AuthFlowIT` (real HTTP, `TestRestTemplate` against a random port, not
  mocked) covers: signup, login, multi-org selection, switch-organization
  (including rejection for a non-member org), refresh rotation +
  rejection on reuse, logout, wrong-OTP rejection without consuming the
  real code, OTP replay rejection, and rate limiting. Verified for real
  against `docker compose` Postgres in this sandboxed session per the
  same method as the foundation slice (see `TEST_VERIFICATION.md`), then
  `AbstractIntegrationTest` reverted to the checked-in Testcontainers
  version.
- **Next session should start with**: `docs/context.md` "Immediate next
  step" — Module 1 (Customers) + Module 3 (Billing) + Module 12's core
  loop, now unblocked since real auth exists.

## 2026-09-13 — Foundation: monorepo skeleton, first migration, tenant isolation

- First application code in the repository. Decided V1's "one combined
  backend service" (docs/mvp-scope.md) is **Java Spring Boot only** — no
  Python in V1; AI extraction (text/photo) will call a hosted multimodal
  LLM API directly from Java rather than running a separate pipeline.
  Documented in `docs/tech-stack.md` and `docs/context.md`.
- Added `services/core` (Spring Boot 3, Java 21, Maven + wrapper) and
  `packages/database/migrations/V1__tenancy_and_identity.sql`: the
  `organizations`/`users`/`roles`/`permissions`/`organization_members`
  schema from `docs/database.md`, seeded with the four default roles and
  their permission mappings from `docs/permissions.md`.
- Implemented Row-Level Security (ADR 0013) as a real, tested guarantee
  rather than an assertion: `organization_members` has RLS enabled and
  `FORCE`d, keyed on a `current_setting('app.current_org_id', true)`
  session variable compared as text (not cast to uuid — a Postgres
  custom-GUC subtlety where `RESET`/an unset value can read back as an
  empty string rather than NULL, which a uuid cast would throw on; text
  comparison degrades safely to zero rows in every case instead). The
  restricted `nexora_app` role (non-superuser, so RLS actually applies to
  it) is created by the migration; Flyway itself still runs as the
  elevated `nexora_admin` role via a separate connection.
- `TenantContext` (a ThreadLocal) + `TenantAwareDataSource` (a
  `DataSource` decorator that stamps the current organization id onto
  every JDBC connection at checkout, via `set_config(..., is_local =>
  false)`) apply the tenant context at the connection level rather than
  via an `@Transactional`-matching AOP aspect — the latter was the first
  approach tried, but Spring Data JPA repository proxies carry
  `@Transactional` on the generated implementation class, not the
  interface, and a plain `TransactionTemplate` isn't annotation-driven at
  all, so an annotation-matching aspect would silently miss both.
- `TenantIsolationIT` proves the guarantee end-to-end (not by inspection):
  a `findAll()` with no organization filter sees only the current
  tenant's row, sees zero rows for another tenant, and zero rows with no
  context set at all. `RoleSeedDataTest` confirms the seed data matches
  `docs/permissions.md`.
- This sandboxed session's Bash tool can drive the `docker` CLI but not
  the raw Docker Engine API Testcontainers needs (every Testcontainers
  call got a stub 400 response regardless of which named pipe it tried).
  The checked-in tests use Testcontainers (correct for a real dev machine
  or CI); this session verified them for real by temporarily pointing the
  same test classes at a `docker compose`-started Postgres instead, then
  reverted to the Testcontainers version once both tests passed for real
  — see `services/core/TEST_VERIFICATION.md` for the full transcript.
- `docker-compose.yml` + `.env.example` for local dev; `GET
  /api/v1/health` confirms the app boots and reaches the database.
- **Next session should start with**: `docs/context.md` "Immediate next
  step" — Module 1 (Customers) + Module 3 (Billing) + Module 12's core
  loop, plus real phone+OTP authentication (needed to populate
  `TenantContext` from an actual logged-in user) and the `apps/web`
  scaffold.

## 2026-07-19 — CTO critique pass: closing the gaps between vision and plan

- Ran a deliberately harsh internal review of everything documented so
  far and fixed every gap that was fixable on paper (the ones that
  actually need real users, capital, and time — like whether this
  becomes a large company — were named honestly as not fixable by
  documentation).
- The single highest-leverage fix: [docs/mvp-scope.md](mvp-scope.md).
  The project had accumulated a full target architecture (three
  services, Kubernetes, Elasticsearch, a native mobile app, a Tally
  local agent) before any real business had touched the product. V1 is
  now explicitly smaller, with a five-point validation gate that has to
  pass before Phase 2 starts.
- Closed a real self-contradiction: near-certain product-alias merges
  were being auto-committed with only a weekly digest, which silently
  violated `PRODUCT_PRINCIPLES.md`'s "no silent financial writes" rule
  (merging products merges their stock ledgers). Fixed to an immediate,
  undoable inline confirmation instead.
- Closed a real strategy contradiction: earlier planning said Tally is a
  companion ([ADR 0006](adr/0006-tally-integration.md)); later planning
  said the explicit goal is making Tally unnecessary. [ADR
  0012](adr/0012-tally-adoption-sequencing.md) resolves this: companion
  framing through Phase 2, independent accounting built for its own
  value in Phase 3+, Tally independence as an earned outcome rather than
  a stated near-term objective.
- Added the gaps that were missing outright: GST invoice correctness as
  a Phase 1 (not Phase 3) concern
  ([gst-compliance.md](gst-compliance.md)), an explicit offline/
  connectivity decision instead of a silent cloud-only assumption
  ([offline-strategy.md](offline-strategy.md)), a UI/UX philosophy
  matched to the actual buyer instead of a Stripe/Notion reference
  ([ui-ux-guidelines.md](ui-ux-guidelines.md)), an AI cost model so
  pricing isn't set blind ([monetization.md](monetization.md)), Postgres
  Row-Level Security as a real tenant-isolation guarantee instead of
  "developers remember to filter" ([ADR
  0013](adr/0013-row-level-security.md)), and a price-consistency signal
  in product matching that was missing entirely.
- Landed as 13 separate commits, each independently reviewable.
- **Next session should start with**: [docs/mvp-scope.md](mvp-scope.md)
  in full, then [docs/context.md](context.md) "Immediate next step."

## 2026-07-19 — Catalog Intelligence: self-growing catalog design

- Worked out the product identity resolution problem in depth: how the
  catalog builds itself from supplier bills, sales invoices, photos,
  voice, and handwritten notes instead of requiring upfront data entry,
  and how near-duplicate product names get merged (or correctly kept
  separate) without either silent duplication or repeated nagging.
- Added `docs/product-catalog-lifecycle.md` (the full design, ten
  worked scenarios, the four-signal matching model, three confidence
  tiers) and `docs/adr/0009-product-catalog-identity-resolution.md`
  (the accepted decision: AI extraction stages into
  `candidate_products`, never writes `products` directly).
- Updated `docs/database.md`, `docs/ai-architecture.md`,
  `docs/pricing-engine.md`, `PROJECT_SPEC.md` (new Module 12),
  `docs/features.md`, `docs/roadmap.md` (Module 12's core loop scheduled
  in Phase 1, not after it), `docs/glossary.md`, `MASTER_INDEX.md`,
  `docs/security.md`, and `docs/testing-strategy.md` to match.
- Landed as 10 separate commits, each independently reviewable.
- **Next session should start with**: [docs/context.md](context.md)
  "Immediate next step" — the monorepo skeleton, first migration, and
  Module 1/3/12 built together.

## 2026-07-19 — Documentation and architecture baseline

- Created the full documentation baseline: `README.md`, `MASTER_INDEX.md`,
  `PROJECT_SPEC.md`, `PRODUCT_PRINCIPLES.md`, `AI_RULES.md`, and the
  `docs/*` engineering, project-memory, and ADR files.
- No application code written yet. This session established the product
  vision, module breakdown, system architecture (monorepo:
  `apps/web`/`apps/mobile`, `services/core`, `services/ai`,
  `services/connectors`), database schema plan, API conventions, AI
  pipeline design, and the Tally/connector integration strategy.
- Repository initialized and pushed to a private GitHub repo
  (`Jagpreet-Tuteja/Nexora`).
- **Next session should start with**: [docs/context.md](context.md)
  "Immediate next step" — scaffolding the monorepo and the first database
  migration.
