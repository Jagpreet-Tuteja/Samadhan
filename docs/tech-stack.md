# Tech Stack

Status: proposed, not yet implemented. Revisit as ADRs when a choice is
actually adopted.

**This is the target stack, not the starting stack.** See
[mvp-scope.md](mvp-scope.md) for what V1 actually runs on: one combined
backend service, PostgreSQL only (no Redis/Elasticsearch), web-only (no
mobile), no Tally agent, no Kubernetes. Everything below is where the
architecture grows *into* once the MVP validation gate is cleared, not
what gets built first.

## Frontend
- React + TypeScript — `apps/web`
- React Native — `apps/mobile` (Phase 2)

## Backend
- Java Spring Boot — `services/core` (transactional business logic, REST
  API)
- Python (FastAPI) — `services/ai` (OCR, vision, LLM orchestration,
  speech)

## Data
- PostgreSQL — primary datastore, multi-tenant
- Redis — caching, session/rate-limit state
- Elasticsearch — product/customer/invoice full-text and fuzzy search (AI
  entity resolution relies on this)
- S3-compatible object storage — invoice PDFs, OCR source images,
  supplier documents

## Auth
- JWT access + refresh tokens
- OTP delivery via SMS/WhatsApp Business API
- Google OAuth (secondary)

## AI
- Hosted LLM API (e.g. Claude) for language understanding/structuring
- Dedicated OCR/vision engine for document and image extraction
- Hosted speech-to-text / text-to-speech API for voice input/output

## Infrastructure
- Docker containers per service
- Kubernetes for orchestration at scale (start on a simpler managed
  container platform pre-Series-A; see [deployment.md](deployment.md))
- GitHub Actions for CI/CD
- Grafana + Prometheus for monitoring

## Why these choices

- Java Spring Boot for `services/core`: mature ecosystem for
  transactional, auditable, high-reliability financial logic; strong
  typing reduces the class of bugs that matter most in a billing system.
- Python for `services/ai`: the AI/ML ecosystem (LLM SDKs, OCR/vision
  libraries, speech APIs) is Python-first.
- Separating the two lets each scale, deploy, and evolve independently —
  see [architecture.md](architecture.md).

## V1 implementation notes (as actually built)

- **V1's combined backend service is Java Spring Boot only** — decided
  2026-09-13. V1's AI scope (Module 4: text + photo input) calls a hosted
  multimodal LLM API directly from Java over HTTP; there is no local
  Python OCR/vision pipeline in V1, so Python isn't needed until AI work
  grows beyond that (voice, embeddings, background catalog clustering —
  see [ai-architecture.md](ai-architecture.md)), at which point
  `services/ai` splits out per the target architecture.
- **Flyway** manages migrations (`packages/database/migrations/`), run
  via a separate, elevated connection distinct from the application's own
  restricted runtime connection — see
  [adr/0013-row-level-security.md](adr/0013-row-level-security.md) for
  why that split matters.
- **Testcontainers** backs every test that touches Postgres (RLS behavior
  can't be verified against an in-memory database).
