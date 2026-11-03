# Nexora

**The AI Business Operating System**

Nexora is an AI-powered business operating system that becomes the digital
brain of small and medium businesses — hardware stores, medical stores,
grocery stores, electrical shops, wholesalers, manufacturers, service
businesses, restaurants, and more. Instead of forcing owners to learn
another piece of accounting software, Nexora understands natural language,
voice, photos, PDFs, and spreadsheets, and turns them into structured
business actions: invoices, price updates, stock movements, purchase
orders, and reports.

Nexora does not try to replace every existing tool on day one. It is
designed to **connect to and interoperate with** the software businesses
already run on — starting with Tally, and extending to Busy, Marg, Zoho
Books, QuickBooks, and plain Excel/CSV workflows — so a business can adopt
Nexora gradually without losing its existing data or habits.

## Current Phase

Nexora has moved from documentation-only into early implementation. The
tenancy/identity foundation (`services/core`, the first database
migration, Row-Level Security), phone+OTP auth, Module 1 (Customers),
Module 2 (Inventory, with manual retail pricing) and Module 3 (Billing, GST
invoices) are built; Catalog Intelligence and
the remaining modules are next. This repository remains the canonical source
of truth for the product vision, system design, and engineering plan —
see [docs/context.md](docs/context.md) for exactly where things stand and
what's next.

## Required Reading Before Implementation

Before any implementation session, read these files in order:

- [MASTER_INDEX.md](MASTER_INDEX.md)
- [PROJECT_SPEC.md](PROJECT_SPEC.md)
- [AI_RULES.md](AI_RULES.md)
- [PRODUCT_PRINCIPLES.md](PRODUCT_PRINCIPLES.md)
- [docs/context.md](docs/context.md)
- [docs/roadmap.md](docs/roadmap.md)
- [docs/features.md](docs/features.md)
- [docs/decisions.md](docs/decisions.md)
- Latest records in [docs/adr](docs/adr)

## Documentation Index

See [MASTER_INDEX.md](MASTER_INDEX.md) for the full, maintained index.

## Getting Started (local development)

Requires Docker and a JDK 21+ (the Maven wrapper fetches Maven itself).

```bash
cp .env.example .env      # defaults work as-is for local dev
docker compose up -d      # starts Postgres
cd services/core
./mvnw spring-boot:run    # applies migrations, starts the API on :8080
curl http://localhost:8080/api/v1/health
```

`./mvnw test` runs the test suite, including Testcontainers-backed
integration tests that verify Row-Level Security against a real,
disposable Postgres container — see
[adr/0013-row-level-security.md](docs/adr/0013-row-level-security.md).

## Product Principles

Nexora is not another accounting app, not another billing app, and not
another ERP clone. It is a conversational layer over a business's data —
customers, inventory, pricing, suppliers, and money — that removes
repetitive manual work. See [PRODUCT_PRINCIPLES.md](PRODUCT_PRINCIPLES.md).

## Technical Direction

Planned architecture is a monorepo with:

- `apps/web` — React web dashboard.
- `apps/mobile` — React Native app (Phase 2).
- `services/core` — Java Spring Boot backend (customers, inventory,
  billing, suppliers, expenses, GST).
- `services/ai` — Python AI services (OCR, vision, LLM orchestration,
  speech-to-text/text-to-speech, embeddings).
- `services/connectors` — integration adapters (Tally, Busy, Marg, Zoho
  Books, QuickBooks, WhatsApp Business, email, CSV/Excel).
- `docs/*` — durable project memory and specifications.

See [docs/tech-stack.md](docs/tech-stack.md) and
[docs/architecture.md](docs/architecture.md) for details, and
[docs/integrations.md](docs/integrations.md) for how Nexora talks to Tally
and other billing software.

## Documentation Policy

Documentation is part of the product. Every completed unit of work must
update the relevant documentation so another engineer or AI agent can
continue from the repository alone, without relying on chat history. See
[AI_RULES.md](AI_RULES.md).

## License

Proprietary. All rights reserved. This repository and its contents are not
licensed for external use, copying, or distribution.
