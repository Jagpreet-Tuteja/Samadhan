# 0004 — Permissions

Status: Accepted
Date: 2026-07-19

## Context

Businesses have multiple staff roles with clearly different needs:
owners need full control, accountants need financial visibility without
price-approval rights, cashiers need to bill without seeing business-wide
reports.

## Decision

Role-based access control (RBAC) scoped per organization, with fixed
default roles (Owner, Manager, Accountant, Staff/Cashier) at launch and
custom roles reserved for a later, higher subscription tier. See
[docs/permissions.md](../permissions.md).

## Consequences

- Enforcement must live server-side in `services/core`; the frontend only
  reflects it.
- AI-drafted actions still require the acting user to hold the
  underlying permission — the AI layer is never a permissions bypass.
- Every permission-gated action is audit-logged.
