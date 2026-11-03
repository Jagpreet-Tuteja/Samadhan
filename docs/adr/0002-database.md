# 0002 — Database

Status: Accepted
Date: 2026-07-19

## Context

Nexora needs a relational, ACID-compliant datastore for financial and
inventory data where correctness and auditability matter more than
schema flexibility, plus full-text/fuzzy search for AI entity resolution
(matching extracted product/customer names against the catalog).

## Decision

PostgreSQL as the primary datastore, Redis for caching/session state, and
Elasticsearch for full-text/fuzzy search. See [docs/database.md](../database.md)
for the schema plan.

## Consequences

- Strong consistency for money/inventory data; no eventual-consistency
  surprises in ledgers.
- Elasticsearch must be kept in sync with PostgreSQL (via change events or
  a sync job) — this sync mechanism needs its own design before Module 1
  ships.
- Multi-tenancy is enforced via `organization_id` row scoping (see
  [ADR 0007](0007-multi-tenancy.md)), not separate databases per tenant.
