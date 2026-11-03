# 0007 — Multi-Tenancy

Status: Accepted
Date: 2026-07-19

## Context

Nexora serves many independent businesses from a shared platform. The
tenancy model must guarantee data isolation while keeping operational
overhead (deployments, migrations, scaling) manageable for a small team.

## Decision

Single shared deployment with `organization_id` row-level scoping on
every business-owned table, rather than a database-per-tenant or
schema-per-tenant model. See [docs/database.md](../database.md).

## Consequences

- Every query in `services/core` must be scoped by `organization_id`;
  this is enforced in code and covered by automated tests (see
  [docs/security.md](../security.md)), since a missed scope is a
  cross-tenant data leak.
- Operationally simpler: one set of migrations, one set of infrastructure
  to scale and monitor, versus thousands of per-tenant databases.
- Large tenants can be moved to dedicated read replicas later without
  changing the logical model (see [docs/architecture.md](../architecture.md)
  scaling strategy).
- Revisit only if a specific enterprise customer requires physical data
  isolation (e.g. dedicated infrastructure) as a contractual requirement.
