# 0005 — AI Integration Architecture

Status: Accepted
Date: 2026-07-19

## Context

Nexora's core differentiator is AI-native input (Modules 4, 5, 6, 10),
but the transactional backend must remain deterministic, auditable, and
independently testable regardless of AI provider changes or outages.

## Decision

`services/ai` is a separate Python service from `services/core` (Java).
The AI layer only ever produces drafts with confidence scores;
`services/core` performs all actual commits, and only after explicit
owner confirmation for anything touching money, price, or stock. See
[docs/ai-architecture.md](../ai-architecture.md).

## Consequences

- `services/core` has zero direct dependency on any LLM/OCR/speech
  provider — it can be developed, tested, and deployed independently.
- Every AI-driven change has a full audit trail
  (`ai_extraction_jobs`, see [docs/database.md](../database.md)):
  raw input, draft, confidence scores, final confirmed values, approver.
- Adds one network hop between AI extraction and commit, an accepted
  cost for the trust guarantee this buys (see
  [PRODUCT_PRINCIPLES.md](../../PRODUCT_PRINCIPLES.md)).
