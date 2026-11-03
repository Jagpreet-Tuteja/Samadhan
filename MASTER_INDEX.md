# Nexora Master Index

This is the first document every engineer or AI agent should read before
working on Nexora.

## Core Project Documents

- [README.md](README.md): Project overview and documentation entry point.
- [PROJECT_SPEC.md](PROJECT_SPEC.md): Canonical product and engineering
  brief — the full module breakdown and vision.
- [AI_RULES.md](AI_RULES.md): Permanent instructions for future AI
  sessions.
- [PRODUCT_PRINCIPLES.md](PRODUCT_PRINCIPLES.md): Product philosophy and
  design principles.
- [docs/mvp-scope.md](docs/mvp-scope.md): **Read this before building
  anything.** The hard line between V1 and everything else, plus the
  validation gate that must pass before Phase 2.
- [docs/future-modules.md](docs/future-modules.md): The long-term
  Business-OS vision, explicitly unscheduled and separated from the
  active spec.

## Project Memory

- [docs/context.md](docs/context.md): Permanent project memory.
- [docs/progress.md](docs/progress.md): Current progress and latest
  session summary.
- [docs/roadmap.md](docs/roadmap.md): Phases, milestones, and backlog.
- [docs/features.md](docs/features.md): Feature register and status.
- [docs/changelog.md](docs/changelog.md): Meaningful project changes.

## Engineering Documents

- [docs/architecture.md](docs/architecture.md): System architecture,
  services, dependencies, and scaling strategy.
- [docs/database.md](docs/database.md): Database model, tables,
  relationships, and migration history.
- [docs/api.md](docs/api.md): API conventions, endpoint plan,
  authentication, and error handling.
- [docs/authentication.md](docs/authentication.md): Authentication model
  and session strategy.
- [docs/permissions.md](docs/permissions.md): Authorization model, roles,
  and policy rules.
- [docs/ai-architecture.md](docs/ai-architecture.md): OCR, vision, LLM,
  and speech pipeline design.
- [docs/pricing-engine.md](docs/pricing-engine.md): Price Intelligence
  module design (Module 5).
- [docs/product-catalog-lifecycle.md](docs/product-catalog-lifecycle.md):
  Self-growing catalog and product identity resolution design (Module 12).
- [docs/gst-compliance.md](docs/gst-compliance.md): GST invoice
  correctness (V1) vs. filing/e-invoicing (Phase 3).
- [docs/offline-strategy.md](docs/offline-strategy.md): V1 connectivity
  tradeoff and the sync-safe API design that makes offline-first
  additive later, not a rearchitecture.
- [docs/ui-ux-guidelines.md](docs/ui-ux-guidelines.md): Design principles
  matched to the actual buyer, not a SaaS/design-tool aesthetic.
- [docs/integrations.md](docs/integrations.md): Tally and other
  billing/accounting software connectors.
- [docs/industry-config.md](docs/industry-config.md): How Nexora adapts
  to different industries via configuration.
- [docs/deployment.md](docs/deployment.md): Deployment architecture and
  operational strategy.
- [docs/tech-stack.md](docs/tech-stack.md): Selected technologies and
  reasons.
- [docs/coding-standards.md](docs/coding-standards.md): Coding standards
  and quality expectations.
- [docs/testing-strategy.md](docs/testing-strategy.md): Test layers and
  AI accuracy evaluation approach.
- [docs/security.md](docs/security.md): Data protection, access control,
  and AI-specific security risks.
- [docs/monetization.md](docs/monetization.md): Pricing tiers and
  go-to-market notes.
- [docs/competitor-analysis.md](docs/competitor-analysis.md): Competitive
  landscape and differentiation.
- [docs/glossary.md](docs/glossary.md): Shared product and engineering
  language.

## Decisions

- [docs/decisions.md](docs/decisions.md): High-level decision log.
- [docs/adr](docs/adr): Architecture decision records.

Current ADRs:

- [0001-monorepo.md](docs/adr/0001-monorepo.md)
- [0002-database.md](docs/adr/0002-database.md)
- [0003-authentication.md](docs/adr/0003-authentication.md)
- [0004-permissions.md](docs/adr/0004-permissions.md)
- [0005-ai-integration-architecture.md](docs/adr/0005-ai-integration-architecture.md)
- [0006-tally-integration.md](docs/adr/0006-tally-integration.md)
- [0007-multi-tenancy.md](docs/adr/0007-multi-tenancy.md)
- [0008-deployment.md](docs/adr/0008-deployment.md)
- [0009-product-catalog-identity-resolution.md](docs/adr/0009-product-catalog-identity-resolution.md)
- [0010-mvp-scope-cut.md](docs/adr/0010-mvp-scope-cut.md)
- [0011-offline-first-strategy.md](docs/adr/0011-offline-first-strategy.md)
- [0012-tally-adoption-sequencing.md](docs/adr/0012-tally-adoption-sequencing.md)
- [0013-row-level-security.md](docs/adr/0013-row-level-security.md)
- [0014-v1-java-only-backend.md](docs/adr/0014-v1-java-only-backend.md)

## Current Status

Early implementation. The tenancy/identity foundation
(`services/core`, the first database migration, Row-Level Security) is
built and tested, along with phone+OTP auth, Module 1 (Customers),
Module 2 (Inventory, with manual retail pricing) and Module 3 (Billing).
This
repository still contains the full product specification, system
architecture, database plan, API plan, AI pipeline design, and
integration strategy (Tally and similar billing software). See
[docs/context.md](docs/context.md) for exactly where things stand and
what to build next.
