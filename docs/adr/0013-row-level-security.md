# 0013 — Row-Level Security as Defense-in-Depth for Multi-Tenancy

Status: Accepted
Date: 2026-07-19

## Context

[ADR 0007](0007-multi-tenancy.md) accepted row-level `organization_id`
scoping over database-per-tenant. As originally documented, isolation was
described as "enforced at the query layer... covered by automated
tests" — meaning the guarantee lived entirely in `services/core`
application code remembering to filter every query correctly. For data
that includes GST numbers and financial ledgers across many businesses, a
single missed `WHERE organization_id = ?` in one query path is a
cross-tenant data leak, and relying solely on code discipline and test
coverage to prevent that is weaker than the sensitivity of the data
warrants.

## Decision

Enable PostgreSQL Row-Level Security policies on every tenant-owned
table, scoped to the connection's current organization context, as a
second, independent enforcement layer behind (not instead of) the
existing application-layer scoping in `services/core`. See
[database.md](../database.md).

## Consequences

- A missed `organization_id` filter in application code degrades to a
  query returning zero rows (or erroring), not a cross-tenant leak — RLS
  is a backstop, not a replacement for writing correct queries.
- Adds real setup and testing cost: every new tenant-owned table needs an
  RLS policy defined alongside its migration, and the connection context
  (which organization is "current") needs to be set correctly per
  request in `services/core` — this needs to become a checked step in
  the standard feature workflow (see [AI_RULES.md](../../AI_RULES.md)),
  not an occasional afterthought.
- Slight query performance overhead from policy evaluation, generally
  acceptable given the correctness guarantee it buys; revisit only if
  profiling shows it as an actual bottleneck.
- This does not replace the automated cross-tenant isolation tests
  already expected in [security.md](../security.md) — both layers are
  intentional, independent checks on the same guarantee.
