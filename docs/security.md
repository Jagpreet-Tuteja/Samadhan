# Security

Status: baseline plan for a system that will hold GST numbers, financial
ledgers, and customer data for many businesses.

## Data protection

- Encryption in transit (TLS everywhere) and at rest (database and object
  storage encryption).
- Tenant isolation enforced at two independent layers: application-layer
  `organization_id` scoping in every `services/core` query, backed by
  PostgreSQL Row-Level Security policies at the database level (see
  [adr/0013-row-level-security.md](adr/0013-row-level-security.md)) so a
  missed application-layer filter degrades to an empty result, not a
  cross-tenant leak. Both layers are covered by automated tests, not just
  code review.
- Secrets (API keys, connector credentials, DB credentials) in a managed
  secrets store, never in source control or plain env files committed to
  git.

## Access control

- RBAC per [permissions.md](permissions.md), enforced server-side.
- Re-authentication required for sensitive actions (data export, GST
  number change, user removal).
- Full audit log (`audit_logs`) for every write, including AI-confirmed
  actions, with actor, role, and timestamp.

## AI-specific risks

- Prompt injection via untrusted input (e.g. a WhatsApp message or PDF
  crafted to manipulate the LLM into misclassifying an action) is
  mitigated by never letting the AI layer call a committing endpoint
  directly — every AI output is a draft requiring explicit owner
  confirmation (see [ai-architecture.md](ai-architecture.md)).
- OCR/vision inputs (photos, PDFs) are treated as untrusted user content:
  scanned for malware on upload, and never executed or parsed as
  anything other than image/document data.

## Compliance direction

- GST-related data handling follows applicable Indian data protection and
  tax record-keeping requirements; formal compliance review is a
  pre-launch gate, not a Phase 1 engineering task, but the data model
  (append-only ledgers, audit logs) is designed to support it from the
  start.

## Catalog data and the future cross-tenant dictionary

Product identity resolution (see
[product-catalog-lifecycle.md](product-catalog-lifecycle.md) and
[adr/0009-product-catalog-identity-resolution.md](adr/0009-product-catalog-identity-resolution.md))
learns per-organization only by default: aliases, attributes, and match
history for one tenant are never visible to, or used to influence
matching for, another tenant. A future, opt-in feature to bootstrap a new
organization's catalog from patterns learned across other businesses in
the same industry (see the Phase 4 roadmap note) would need, before it
can be built: aggregation and anonymization that cannot be reversed to
identify a source business or its actual product list/pricing, explicit
opt-in consent per organization, and exclusion of anything
customer/pricing-specific from whatever is shared — only generic
category/attribute/naming patterns. Treat this as a privacy design task
in its own right, not a natural extension of the per-tenant matching
pipeline.

## Connector credentials

Tally and other connector credentials (see [integrations.md](integrations.md))
are stored per-organization, scoped to that organization only, and never
shared across tenants even if two businesses happen to use the same local
Tally installation pattern.
