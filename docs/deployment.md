# Deployment

Status: design stage.

**V1 deploys as a single service + managed Postgres**, per
[mvp-scope.md](mvp-scope.md) — no separate `services/ai`,
`services/connectors`, Kubernetes, or CI complexity beyond a basic
build/deploy pipeline. Everything below describes the target topology
this grows into after the MVP validation gate passes, not the starting
setup.

## Environments

- `local` — docker-compose running all services for development.
- `staging` — mirrors production topology at lower scale, used for
  pre-release verification.
- `production` — multi-tenant live environment.

## Planned topology

- Each service (`apps/web`, `services/core`, `services/ai`,
  `services/connectors`) is containerized and deployed independently.
- Start on a managed container platform (e.g. a single cloud provider's
  managed Kubernetes or a simpler PaaS) to minimize ops overhead
  pre-revenue; move to a more elaborate Kubernetes setup only once scale
  demands it (see [architecture.md](architecture.md) scaling strategy).
- Managed PostgreSQL and managed Redis rather than self-hosted, for
  reliability at small-team scale.
- CI/CD via GitHub Actions: lint + typecheck + test on every PR; deploy
  on merge to `main` after checks pass.

## The Tally connector's special case

Because Tally typically runs on a business's local machine with no public
network exposure, the Tally connector is the one component that is not
purely cloud-deployed — see [integrations.md](integrations.md) for the
local-agent model.

## Backups & recovery

- Automated daily database backups with point-in-time recovery.
- Object storage (invoice PDFs, source documents) versioned and backed up
  independently of the database.

See [adr/0006-deployment.md](adr/0006-deployment.md) once a concrete
provider decision is made.
