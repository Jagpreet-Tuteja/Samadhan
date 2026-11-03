# 0010 — MVP Scope Cut

Status: Accepted
Date: 2026-07-19

## Context

Prior planning specified a target architecture — separate `services/core`
and `services/ai`, PostgreSQL plus Redis plus Elasticsearch, a native
mobile app, a Tally local agent, Kubernetes — before a single real
business had used any part of the product. Continuing to build toward
that target architecture as the starting point risks spending months on
infrastructure the project has no evidence yet that it needs, which is
the same premature-architecture failure mode common to early-stage
products that plan like they've already found product-market fit.

## Decision

Cut V1 down to what's specified in [docs/mvp-scope.md](../mvp-scope.md):
one combined backend service, PostgreSQL only, web app only, no Tally
local agent (manual CSV bridge instead), no Kubernetes. The full target
architecture in [architecture.md](../architecture.md),
[tech-stack.md](../tech-stack.md), and [deployment.md](../deployment.md)
remains the documented destination, reached only after the MVP
validation gate passes — it is not deleted or deprioritized, only
resequenced to come after real usage justifies it rather than before.

## Consequences

- V1 ships faster and cheaper, at the cost of some components (search,
  caching, service separation) needing real migration work later instead
  of being built correctly the first time — an accepted tradeoff, since
  "correct for a scale that may never arrive" is a worse failure mode
  than "needs a migration once real scale arrives."
- Every target-state document now explicitly states it describes the
  destination, not the starting point, so future sessions don't
  mistake the target architecture for what to build first.
- The validation gate in [docs/mvp-scope.md](../mvp-scope.md) is the
  explicit trigger for starting the migration toward target
  architecture — this is a decision to revisit with evidence, not on a
  fixed calendar.
