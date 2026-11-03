# 0008 — Deployment Platform (Initial)

Status: Accepted
Date: 2026-07-19

## Context

Pre-revenue, small team: minimizing operational overhead matters more
than maximizing infrastructure sophistication. Full Kubernetes is
significant ops burden for a handful of services at low initial scale.

## Decision

Start on a managed container platform (managed Kubernetes or a simpler
PaaS on a single cloud provider) with managed PostgreSQL and managed
Redis, rather than a self-managed Kubernetes cluster, for the initial
launch. See [docs/deployment.md](../deployment.md).

## Consequences

- Faster initial setup, lower ops burden, at the cost of some
  infrastructure flexibility.
- Revisit and likely move to a more elaborate Kubernetes setup once scale
  (tenant count, request volume) justifies the added operational
  investment — tracked as a future ADR, not a Phase 1 concern.
- The Tally connector's local-agent component (see
  [ADR 0006](0006-tally-integration.md)) is deployed and distributed
  separately from this cloud topology.
