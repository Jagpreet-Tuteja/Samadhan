# Permissions

Status: design stage.

## Model

Role-based access control (RBAC), scoped per organization, with a fixed
set of default roles plus room for custom roles on higher subscription
tiers.

## Default roles

- **Owner** — full access, including billing/subscription management,
  user management, and connector configuration (Tally, etc.).
- **Manager** — full operational access (billing, inventory, customers,
  suppliers, expenses, reports) but no subscription/billing-account
  changes and no user management beyond staff invites.
- **Accountant** — read/write on billing, expenses, and reports; read-only
  on inventory; no price-approval rights.
- **Staff/Cashier** — can create invoices and quotations using existing
  prices and stock; cannot approve price changes, edit product catalog,
  or view business-wide financial reports.

## Implemented notes

- `org.manage_settings` (added in `V4__billing.sql`, Owner only) gates
  the organization's GST profile, since it changes the legal format of
  every invoice that follows.
- Setting a catalog price, or billing at a price other than the catalog
  price, needs `billing.approve_price_change` (Owner, Manager). Staff
  bill at catalog prices only.
- Staff hold `billing.create` but not `billing.read`: they can fetch a
  single invoice (to reprint what they issued) but not list billing
  history.

## Enforcement

- Enforced server-side in `services/core` on every request — the
  frontend hides controls the user can't use, but the API is the source
  of truth for authorization.
- AI-drafted actions (Module 4, Module 5) still require the acting user
  to hold the permission for the underlying action — the AI layer never
  grants elevated access.
- Every permission-gated action is recorded in `audit_logs` with the
  acting user and role at the time of the action.

See [adr/0004-permissions.md](adr/0004-permissions.md).
