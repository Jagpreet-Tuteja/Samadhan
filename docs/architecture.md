# Architecture

## Overview

This document describes the **target architecture** — where the system
grows once real usage justifies the added complexity. It is not the
starting point. See [mvp-scope.md](mvp-scope.md): V1 runs as one combined
backend service against PostgreSQL only, with no separate AI service, no
connectors service, no Elasticsearch/Redis, and no Kubernetes. The split
described below is the destination, reached deliberately, not the day-one
design — building it before there's a real scaling reason is exactly the
premature-architecture risk this document used to invite by omission.

Nexora, at target scale, is a monorepo containing a web frontend, a core
business-logic backend, a Python AI services layer, and a connector layer
for external systems (Tally and similar software). The AI layer is
deliberately separated from the core backend so that AI models can be
swapped, retrained, or scaled independently of transactional business
logic.

```
                         +-------------------+
                         |     apps/web      |  React dashboard
                         |   apps/mobile     |  React Native (Phase 2)
                         +---------+---------+
                                   |
                                   v
                       +-----------------------+
                       |    services/core      |  Java Spring Boot
                       |  (customers, billing,  |  REST API, business
                       |   inventory, suppliers,|  rules, GST, ledgers
                       |   expenses, users)     |
                       +-----+-------------+----+
                             |             |
                             v             v
                   +-----------------+  +---------------------+
                   |  services/ai    |  | services/connectors |
                   |  (Python)       |  | (Tally, Busy, Marg,  |
                   |  OCR, vision,   |  |  Zoho, QuickBooks,   |
                   |  LLM, voice     |  |  WhatsApp, CSV)      |
                   +-----------------+  +---------------------+
                             |             |
                             v             v
                   +----------------------------------+
                   |     PostgreSQL + Redis + S3       |
                   +----------------------------------+
```

## Why this split

- `services/core` owns the transactional data model and is the single
  source of truth. It never calls an LLM directly — it only accepts
  structured commands.
- `services/ai` turns unstructured input (text, voice, photo, PDF,
  WhatsApp message) into a structured, confidence-scored proposal, and
  calls back into `services/core` only after the owner confirms.
- `services/connectors` isolates every external-system integration (Tally
  XML interface, Busy/Marg APIs, Zoho/QuickBooks APIs, WhatsApp Business
  API) behind one adapter interface, so `services/core` never has
  system-specific code in it. See [integrations.md](integrations.md).

## Monorepo layout (planned)

```
apps/
  web/                  React + TypeScript dashboard
  mobile/                React Native app (Phase 2)
services/
  core/                  Java Spring Boot — REST API, domain services
  ai/                    Python (FastAPI) — OCR/vision/LLM/speech pipelines
  connectors/            Tally / Busy / Marg / Zoho / QuickBooks / WhatsApp adapters
packages/
  shared-types/          Generated/shared API types for web + mobile
  database/               Schema, migrations
docs/                    Durable project memory (this directory)
```

### What exists today in `services/core`

One package per domain under `com.nexora.core`, each with its own
entities, repositories (every method filters on `organizationId`, behind
RLS), service, DTOs and controller:

- `tenancy`, `security`, `web` — shared plumbing (tenant context + RLS
  session stamping, JWT auth, the response/error envelope).
- `identity`, `auth` — organizations, users, roles, phone+OTP login.
- `customers` — Module 1.
- `inventory` — Module 2: categories, products, auto-SKU, the single
  default warehouse, the stock ledger.
- `pricing` — current retail price + append-only price history. Kept
  separate from `inventory` because Module 5 (Price Intelligence) will
  write here through its diff → approve flow, under a different
  permission (`billing.approve_price_change`) than catalog edits.
- `gst` — pure GST logic with no Spring or database dependency (GSTIN
  validation, `GstCalculator`), so it can be fixture-tested exhaustively.
- `organization` — the organization's own settings (GST profile).
- `billing` — Module 3: invoices, which write through `inventory`
  (sale movements) and `customers` (ledger debits) in one transaction.

## Multi-tenancy

Nexora is multi-tenant from day one: every table under a business is
scoped by `organization_id`. A single Nexora deployment serves many
businesses; a business's data is never visible to another tenant. See
[database.md](database.md) and
[adr/0007-multi-tenancy.md](adr/0007-multi-tenancy.md).

## Request flow example — AI billing

1. Owner sends a WhatsApp message, voice note, or types "10 Cement Bags, 5
   PVC Panels" in the app.
2. `services/connectors` (for WhatsApp) or `apps/web` (for direct input)
   forwards the raw input to `services/ai`.
3. `services/ai` runs the appropriate pipeline (speech-to-text if voice,
   OCR if image/PDF, then LLM extraction) and produces a structured
   `InvoiceDraft` with confidence scores per line item, matched against
   the tenant's product catalog via `services/core`.
4. `apps/web` (or WhatsApp reply) shows the draft to the owner for
   confirmation, highlighting any low-confidence or ambiguous matches
   ("Which Rahul?").
5. On confirmation, `apps/web` calls `services/core` to commit the
   invoice, update inventory, update the customer ledger, and generate
   the PDF.
6. If Tally sync is enabled, `services/core` emits an event that
   `services/connectors` picks up to push the equivalent voucher into
   Tally.

## Scaling strategy

- Stateless `services/core` and `services/ai` instances behind a load
  balancer; horizontal scaling per service.
- `services/ai` is the most compute-intensive layer (OCR/vision/LLM) and
  scales independently, with request queuing for bulk price-update jobs.
- Redis for caching product/price lookups used heavily by the AI matching
  step.
- PostgreSQL with tenant-aware indexing; heavy tenants can be moved to
  dedicated read replicas as volume grows.
- Object storage (S3-compatible) for invoice PDFs, supplier PDFs/photos,
  and OCR source images.

See [deployment.md](deployment.md) for infrastructure and
[tech-stack.md](tech-stack.md) for the selected technologies.
