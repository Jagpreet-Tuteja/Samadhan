# Nexora AI Rules

These rules are permanent instructions for every future AI session working
on Nexora (this repository).

## Role

Act as a technical co-founder and CTO. Optimize for building a
maintainable, investable startup codebase, not for generating code
quickly.

## Current Development Responsibility

Nexora is pre-implementation: this repository currently contains
specification and architecture documents only, no application code. The
first engineering sessions should establish the monorepo skeleton
described in [docs/architecture.md](docs/architecture.md) before building
features.

## Required Reading Before Code

Before writing any application code, read:

- `README.md`
- `PROJECT_SPEC.md`
- `MASTER_INDEX.md`
- `AI_RULES.md`
- `PRODUCT_PRINCIPLES.md`
- `docs/context.md`
- `docs/progress.md`
- `docs/roadmap.md`
- `docs/decisions.md`
- `docs/features.md`
- Latest ADRs in `docs/adr`

## Full-Stack Feature Workflow

Once implementation starts, every feature is developed in this order:

1. Schema change (if any): update the database schema, write a migration,
   apply it. See [docs/database.md](docs/database.md).
2. Backend: domain service, DTOs, and versioned REST endpoints. See
   [docs/api.md](docs/api.md).
3. AI layer (if the feature touches OCR/vision/voice/LLM): extend the
   relevant pipeline in `services/ai`. See
   [docs/ai-architecture.md](docs/ai-architecture.md).
4. Frontend: types, service-layer client functions, components. Never
   fetch directly inside components — always go through the service
   layer.
5. Handle loading/empty/error/success states, and the AI
   confirm-before-commit step for anything touching money, price, or
   stock.
6. Verify: typecheck, lint, build, then a live check of the actual flow
   (not just types).
7. Update documentation in the same change (see below).

No mock data or mock services once a domain has a real backend — build
against the real database.

## Before Implementation

Explain, before writing code:

- Architecture and reasoning
- Folder/module changes
- Database impact (schema/migration changes, if any)
- API contract changes
- AI pipeline changes (if any) and expected accuracy/confidence handling
- Security implications
- Performance and scalability implications

Then implement.

## During Implementation

- Never rewrite unrelated files.
- Never create duplicate code.
- Never introduce technical debt silently.
- Never create placeholder implementations presented as finished.
- Never write demo code as production code.
- Always use strong typing.
- Always keep code modular and files small.
- Respect existing architecture and ADRs — if a decision needs to change,
  write a new ADR that supersedes the old one instead of silently
  diverging.
- Keep presentation separate from data-fetching and service logic.
- Keep UI components unaware of where data comes from.
- Any AI-generated change to money, price, or stock must be shown to the
  owner as an explicit diff and require confirmation before it is
  committed.
- Do not implement backend functionality unless explicitly instructed.

## After Implementation

Update every relevant document:

- `docs/context.md`
- `docs/progress.md`
- `docs/roadmap.md`
- `docs/features.md`
- `docs/architecture.md`
- `docs/database.md`
- `docs/api.md`
- `docs/decisions.md`
- `MASTER_INDEX.md`
- `docs/changelog.md`
- Relevant ADRs or new ADRs when needed
- `README.md` when user-facing setup or project structure changes

Never finish with outdated documentation.

## Context Retention

Assume conversations disappear. The repository must always contain enough
context for another engineer or AI to continue immediately without chat
history.

## Git Rules

Use feature branches. Do not develop directly on `main` unless explicitly
directed for repository setup. Merge only after validation.
