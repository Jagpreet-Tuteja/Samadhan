# API

Status: authentication, Module 1 (Customers), Module 2 (Inventory,
with manual retail pricing) and Module 3 (Billing) are implemented — see "Implemented
endpoints" below. Everything else is still
design stage / planned.

## Conventions

- All endpoints are versioned under `/api/v1/*`.
- JSON envelope: `{ "data": ..., "meta": {...} }` on success,
  `{ "error": { "code": "...", "message": "..." } }` on failure.
- Every request is authenticated (JWT bearer token) and scoped to the
  caller's `organization_id` — no endpoint accepts a client-supplied
  tenant ID.
- Mutating endpoints that affect money, price, or stock require an
  explicit `confirm: true` flag when called from an AI-drafted flow; a
  draft can be fetched and inspected before confirmation.
- Pagination via `?cursor=` + `limit`, not offset, for high-volume
  resources (invoices, stock ledger).

## Implemented endpoints

Authentication (adr/0003-authentication.md; see
`com.nexora.core.auth.AuthController`, tested end-to-end by
`AuthFlowIT`):

- `POST /api/v1/auth/otp/request` `{phone}` — issues a 6-digit OTP,
  rate-limited to 5 requests per phone per 15 minutes.
- `POST /api/v1/auth/otp/verify` `{phone, code}` — one of three
  responses: `authenticated` (tokens, single-org existing user),
  `signup_required` (unrecognized phone, returns a short-lived ticket),
  or `select_organization_required` (multi-org user, ticket + org list).
- `POST /api/v1/auth/signup` `{ticket, name, businessName}` — creates
  the organization, its owner user, and the owner membership in one
  atomic transaction; returns tokens.
- `POST /api/v1/auth/select-organization` `{ticket, organizationId}` —
  completes login for a multi-org user.
- `POST /api/v1/auth/switch-organization` `{organizationId}`
  (authenticated) — moves an already-logged-in user to another
  organization they belong to, without a fresh OTP.
- `POST /api/v1/auth/refresh` `{refreshToken}` — rotates; the token just
  replaced is rejected if reused.
- `POST /api/v1/auth/logout` `{refreshToken}` — revokes it.

All other endpoints require `Authorization: Bearer <accessToken>`;
missing/invalid tokens get `401 {"error":{"code":"unauthorized",...}}`
from `com.nexora.core.security.RestAuthenticationEntryPoint`, and
insufficient permissions get `403 {"error":{"code":"forbidden",...}}`.

Module 1 — Customer Management (docs/mvp-scope.md; see
`com.nexora.core.customers.CustomerController`, tested by
`CustomerApiIT`), gated by `customers.read`/`customers.write`
(docs/permissions.md):

- `POST /api/v1/customers` `{name, phone?, address?, gstNumber?,
  creditLimitMinorUnits?, notes?}` — `409` on a duplicate phone within
  the same organization.
- `GET /api/v1/customers?search=&page=&size=` — matches name (partial,
  case-insensitive) or phone (substring); plain page/size pagination for
  V1 (customers aren't the "high-volume" case the cursor convention
  below targets — revisit if that changes).
- `GET /api/v1/customers/{id}` — includes `balanceMinorUnits` (the sum
  of the customer's ledger entries).
- `PATCH /api/v1/customers/{id}` — partial update; a field left out of
  the request body is unchanged (this slice doesn't support explicitly
  clearing an optional field back to blank).
- `POST /api/v1/customers/{id}/ledger-entries` `{entryType,
  amountMinorUnits, note?}` — `entryType` is `opening_balance` or
  `adjustment` only here; `invoice`/`payment`/`credit_note` entries on
  this same table are Module 3's job once it exists.
- `GET /api/v1/customers/{id}/ledger-entries?page=&size=`

Module 2 — Inventory, plus manual retail pricing (docs/mvp-scope.md;
see `com.nexora.core.inventory` and `com.nexora.core.pricing`, tested
by `InventoryApiIT`), gated by `inventory.read`/`inventory.write`,
except setting a price, which needs `billing.approve_price_change`
(Owner/Manager only — docs/permissions.md gives Accountant and Staff no
price-approval rights):

- `POST /api/v1/product-categories` `{name, skuPrefix?, defaultUnit?,
  defaultHsnCode?, defaultGstRateBasisPoints?}` — `skuPrefix` derived
  from the name when omitted ("PVC Panels" → `PVC`); `409` on a duplicate
  name (case-insensitive).
- `GET /api/v1/product-categories` — unpaginated (tens, not thousands).
- `POST /api/v1/products` `{name, categoryId?, sku?, barcode?, unit?,
  hsnCode?, gstRateBasisPoints?, minStockQuantity?}` — `unit`, `hsnCode`
  and `gstRateBasisPoints` fall back to the category's defaults; `unit`
  is `400` if neither supplies one. `sku` is auto-generated when omitted
  (`PVC-0001`, from the category's prefix, else derived from the product
  name), skipping any value a hand-entered SKU already took. `409` on a
  duplicate SKU or barcode. New products start `price_missing`.
- `GET /api/v1/products?search=&categoryId=&pricingStatus=&lowStock=&page=&size=`
  — `search` matches name or SKU (partial, case-insensitive) or barcode
  (exact); `lowStock=true` keeps products at or below their
  `minStockQuantity`. Each product includes `stockOnHand`,
  `retailPriceMinorUnits` (null while `price_missing`) and `lowStock`.
- `GET /api/v1/products/{id}`, `PATCH /api/v1/products/{id}` — PATCH
  semantics as for customers; SKU is not updatable (it's the stable
  identifier printed on barcodes and used by integrations).
- `POST /api/v1/products/{id}/stock-movements` `{movementType,
  quantityDelta, note?}` — `movementType` is `opening_stock` (positive,
  only as the product's first movement, else `409`) or `adjustment`
  (signed, non-zero); sale/purchase movements on the same ledger are
  Module 3's job. Negative on-hand is allowed on purpose (see
  `StockService`).
- `GET /api/v1/products/{id}/stock-movements?page=&size=` — page/size,
  not cursor: per-product history is low-volume; the cursor convention
  below is for org-wide ledger listings once they exist.
- `PUT /api/v1/products/{id}/price` `{amountMinorUnits, note?}` — sets
  the retail price, moves the product to `priced`, and appends to
  `price_history` (`source = manual`, a batch of one, the caller as
  approver). Re-sending the current price writes no history row.
- `GET /api/v1/products/{id}/price-history?page=&size=`

Module 3 — Billing (docs/gst-compliance.md; see `com.nexora.core.billing`,
`com.nexora.core.organization`, `com.nexora.core.gst`, tested by
`BillingApiIT`, `GstCalculatorTest`, `TaxTreatmentTest`):

- `GET /api/v1/organization/tax-profile` (any member) /
  `PUT /api/v1/organization/tax-profile` `{taxScheme, gstNumber?}`
  (`org.manage_settings`, Owner only). `taxScheme` is `regular`,
  `composition` or `unregistered`; the first two need a valid GSTIN
  (checksum verified), from which `stateCode` is derived; `unregistered`
  must not send one. Invoicing returns `400` until this is set.
- `POST /api/v1/invoices` `{id, customerId?, placeOfSupplyStateCode?,
  pricesIncludeTax?, notes?, items: [{productId, quantity,
  unitPriceMinorUnits?}]}` (`billing.create`):
  - `id` is a client-generated UUID; re-sending the same `id` returns
    the already-issued invoice (no second invoice, stock movement or
    ledger debit). An `id` taken by another organization is `409`.
  - Place of supply: request value, else the customer's state, else the
    business's own state (counter sale). Same as the seller's state →
    CGST+SGST; different → IGST. Composition sellers can't sell
    interstate (`400`). Composition/unregistered sellers issue a
    `bill_of_supply` with every tax amount 0.
  - Line price defaults to the product's retail price. A different
    price — or any price for a `price_missing` product — needs
    `billing.approve_price_change` (`403` otherwise) and applies to this
    invoice only. A product with no price and no line price is `400`; a
    tax invoice line for a product with no GST rate is `400`.
  - Number `INV/26-27/0001`, consecutive per organization per financial
    year (April–March, Indian calendar date). Totals are sums of lines,
    rounded to the rupee via `roundOffMinorUnits`.
  - Writes a `sale` stock movement per line and, for a named customer,
    an `invoice` debit on the customer ledger, all in one transaction.
- `GET /api/v1/invoices?customerId=&page=&size=` (`billing.read`) —
  summaries, newest first. Page/size for now; the cursor convention
  above should replace it once invoice volume makes offsets slow.
- `GET /api/v1/invoices/{id}` (`billing.read` or `billing.create`, so
  Staff can reprint what they issued) — full invoice with lines.

Changes to Module 1 in the same slice: customers take an optional
`stateCode` (derived from `gstNumber` when not given), `gstNumber` is
now validated (`400` on a bad checksum), and the ledger endpoint accepts
`entryType: payment` (must be negative). Ledger entries now carry
`referenceType`/`referenceId` (set on `invoice` entries).

## Planned endpoint groups

- `GET/POST /api/v1/suppliers`, `/api/v1/suppliers/{id}`
- `GET /api/v1/invoices/{id}/pdf`
- `GET/POST /api/v1/quotations`, `/api/v1/purchase-orders`,
  `/api/v1/purchase-bills`, `/api/v1/credit-notes`, `/api/v1/debit-notes`,
  `/api/v1/delivery-challans`
- `GET/POST /api/v1/expenses`
- `GET /api/v1/dashboard/summary`, `/api/v1/dashboard/ai-summary`
- `POST /api/v1/ai/extract` — submit text/voice/image/PDF, returns a
  structured draft with confidence scores (does not commit anything).
- `POST /api/v1/ai/chat` — conversational assistant endpoint (Module 10).
- `GET/POST /api/v1/connectors/tally/*` — Tally sync configuration and
  manual sync trigger. See [integrations.md](integrations.md).

## Webhooks (planned)

Outbound webhooks for: invoice created, payment received, low-stock
alert, price-update batch pending approval — so external systems
(including a business's own scripts) can react to Nexora events.

## Error handling

Every AI-facing endpoint returns a `confidence` field per extracted
field, and a `needs_clarification` list when the input is ambiguous
(e.g., multiple customers matching "Rahul") instead of guessing silently.
