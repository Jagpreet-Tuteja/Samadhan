# 0001 — Monorepo Structure

Status: Accepted
Date: 2026-07-19

## Context

Nexora has multiple deployable units from the start: a web frontend, a
core transactional backend, a Python AI service, and a connectors service
for external integrations (Tally and others). These will share types,
evolve together in early stages, and are built by a small team/AI-assisted
workflow where cross-cutting changes (schema + backend + frontend in one
feature) are common.

## Decision

Use a single monorepo with `apps/*` (frontend surfaces), `services/*`
(backend services), `packages/*` (shared code/types/database schema), and
`docs/*` (durable documentation), rather than separate repositories per
service.

## Consequences

- A single PR can span schema, backend, and frontend changes for one
  feature, keeping them atomic and reviewable together.
- Shared types (`packages/shared-types`) avoid drift between backend
  contracts and frontend consumption.
- CI must be scoped per-package to avoid rebuilding/retesting everything
  on every change as the repo grows.
- Revisit if/when a service needs an independent release cadence or a
  separate team boundary that a monorepo actively hinders — not expected
  before Phase 3.
