# Project Context

Permanent project memory. Update this file whenever a durable fact about
the project changes — this is what an engineer or AI agent should read to
understand "where things stand" without chat history.

## What Nexora is

An AI Business Operating System for small and medium businesses (hardware,
medical, grocery, restaurant, wholesale, and more) — see
[PROJECT_SPEC.md](../PROJECT_SPEC.md). Core differentiators: AI-native
input (text/voice/photo/PDF/WhatsApp → structured business data),
industry-agnostic configuration, and interoperability with Tally and
similar existing billing/accounting software rather than forcing
migration.

## Where the project stands

As of 2026-09-13: the foundation slice and real authentication both
exist. `services/core` is a real, running Spring Boot (Java 21)
application — **V1's "one combined backend service" is Java only**, not
Python; AI extraction will call a hosted multimodal LLM API directly from
Java rather than running a separate Python pipeline (see
[tech-stack.md](tech-stack.md)). Two Flyway migrations are applied and
verified: `V1__tenancy_and_identity.sql` (`organizations`, `users`,
`roles`, `permissions` seeded, `organization_members` with Row-Level
Security) and `V2__auth_and_customers.sql` (`otp_challenges`,
`refresh_tokens`, `customers`, `customer_ledger_entries`, and a widened
`organization_members` policy — see [database.md](database.md)). The RLS
guarantee in [adr/0013-row-level-security.md](adr/0013-row-level-security.md)
is proven by automated tests (`TenantIsolationIT`), not just asserted by
the SQL.

Phone+OTP authentication (adr/0003-authentication.md) is real and
tested end-to-end (`AuthFlowIT`): signup, login, multi-org
selection/switching, refresh rotation with per-device revocation, OTP
rate-limiting — see [api.md](api.md) "Implemented endpoints". `TenantContext`
is now populated from real JWTs via `JwtAuthenticationFilter`, not a test
harness. Two non-obvious Spring Security 6 bugs were hit and fixed while
building this — see `services/core/TEST_VERIFICATION.md` if a future
session needs to touch `JwtAuthenticationFilter` or `SecurityConfig`
again, to avoid re-discovering them.

**Module 1 (Customer Management) is also done**: profiles, phone/name
search, and an append-only ledger with a computed balance, all gated by
`customers.read`/`customers.write` (docs/permissions.md) and isolated
per-tenant the same way `TenantIsolationIT` proves for
`organization_members` — see [api.md](api.md) "Implemented endpoints"
and `CustomerApiIT`.

**Module 2 (Inventory, V1 scope) is done as of 2026-11-03**, together
with manual retail pricing — the shared product/price/stock base that
both Billing and Catalog Intelligence need. `V3__catalog_and_inventory.sql`
adds categories, products (auto-SKU `PVC-0001` from a per-org atomic
counter, HSN code, GST rate in basis points, `pricing_status` starting at
`price_missing`), `product_prices` + append-only `price_history`, one
lazily-created default warehouse, and an append-only `stock_ledger`
(on-hand is always a `SUM`, negative allowed on purpose). Code lives in
`com.nexora.core.inventory` and `com.nexora.core.pricing`; see
[api.md](api.md) and `InventoryApiIT`.

**Module 3 (Billing, V1 scope) is done as of 2026-11-03**:
`V4__billing.sql` plus `com.nexora.core.billing`, `.organization` and
`.gst`. The owner sets a GST profile first (`regular`/`composition`/
`unregistered` + GSTIN, checksum-validated); invoicing is refused until
then. Invoices are GST tax invoices (CGST+SGST or IGST by place of
supply) or bills of supply (composition/unregistered, no tax), with
client-generated UUIDs for idempotent retries, server-reserved
`INV/26-27/0001` numbers per financial year, immutable snapshots of the
seller/buyer/product GST details, and — in the same transaction — a
`sale` stock movement per line and an `invoice` debit on the customer
ledger. All tax arithmetic lives in the pure `GstCalculator`, covered by
a hand-worked fixture suite (`GstCalculatorTest`) per the
"100%-correct bar" in [gst-compliance.md](gst-compliance.md). No Module
12 candidate/identity-resolution loop, no invoice PDF, and no frontend
exist yet.

Known testing gap: Surefire's default include patterns don't match
`*IT`, and there's no CI, so a plain `./mvnw test` runs none of the
HTTP integration tests (`AuthFlowIT`, `CustomerApiIT`,
`TenantIsolationIT`, `InventoryApiIT`, `BillingApiIT`) — run them
explicitly with `./mvnw test '-Dtest=*Test,*IT'` (needs Docker; all 68
pass as of 2026-11-03) until a Failsafe/`verify` step is wired in.

## Immediate next step

The backend can now run a shop's core loop (catalog → price → invoice →
stock and customer balance), but nothing reaches a pilot business
without a UI and a printable invoice. Next: `apps/web` scaffolding with
the billing screen and a print-ready invoice view (a GST invoice must
show the legally required fields — seller/buyer GSTIN, HSN, place of
supply, the tax breakup, or the composition declaration on a bill of
supply), following `AI_RULES.md`'s schema → backend → frontend
workflow. Then Module 12's remaining core loop (`candidate_products`,
aliases, live matching, the Product Approval Queue), which feeds Module
4 (AI Billing). Do not start Phase 2 work until the validation gate in
[docs/mvp-scope.md](mvp-scope.md) passes.

## Key decisions already made

See [docs/decisions.md](decisions.md) and [docs/adr](adr) for the full
list. Highlights: V1 is deliberately cut down from the target
architecture (ADR 0010); multi-tenant single deployment enforced at both
the application layer and via Postgres RLS (ADR 0007, ADR 0013); AI layer
is a separate Python service *in the target architecture* — V1 combines
everything into one Java service, calling a hosted LLM API directly for
AI extraction rather than running a local Python pipeline; Tally is a
companion, not a replacement, through at least
Phase 2, with independent accounting built for its own value in Phase 3+
rather than as an explicit "replace Tally" push (ADR 0006, ADR 0012); V1
requires connectivity to bill, with a sync-safe API designed in from day
one so true offline-first can be added in Phase 2 without a rewrite (ADR
0011); AI extraction never writes directly into the canonical `products`
table — every product mention is staged in `candidate_products` and
resolved through tiered-confidence identity matching, with a near-certain
match still surfaced as an undoable confirmation, never fully silent
(ADR 0009).
