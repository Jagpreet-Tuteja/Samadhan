# Coding Standards

Status: baseline expectations, to be refined once implementation starts.

## General

- Strong typing everywhere (TypeScript strict mode; Java; typed Python
  via type hints + mypy in `services/ai`).
- Small, single-responsibility files and functions.
- No commented-out code, no dead code left "just in case".
- No placeholder/demo implementations presented as finished work.

## Backend (`services/core`)
- Domain services own business rules; controllers/route handlers stay
  thin.
- Every mutating endpoint validates permissions server-side (see
  [permissions.md](permissions.md)) regardless of what the frontend
  already checked.
- Money as integer minor units, never floating point (see
  [database.md](database.md)).

## AI services (`services/ai`)
- Every extraction pipeline stage is independently testable with fixture
  inputs (sample images, sample text) and golden expected outputs.
- Prompts are versioned files, not inline strings scattered across the
  codebase, so prompt changes are reviewable diffs.

## Frontend (`apps/web`, `apps/mobile`)
- Components never fetch data directly — always through a service layer.
- Loading/empty/error/success states handled explicitly for every screen
  that touches network data.

## Documentation
- Every feature change updates the relevant `docs/*` file in the same
  change — see [AI_RULES.md](../AI_RULES.md).
