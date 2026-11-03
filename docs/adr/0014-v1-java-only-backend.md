# 0014 — V1's Combined Backend Service Is Java Only

Status: Accepted
Date: 2026-09-13

## Context

[ADR 0010](0010-mvp-scope-cut.md) and [docs/mvp-scope.md](../mvp-scope.md)
decided V1 combines `services/core` and the V1-relevant parts of
`services/ai` into "one backend service," but never specified which
language that service is written in — [docs/tech-stack.md](../tech-stack.md)
still described the target-state split (Java for `services/core`, Python
for `services/ai`) without saying what V1 itself runs on. Starting
implementation required resolving this.

## Decision

V1's combined backend service is **Java (Spring Boot) only**. No Python
in V1. V1's AI scope ([docs/mvp-scope.md](../mvp-scope.md) Module 4: text
and photo input only) is served by calling a hosted multimodal LLM API
(e.g. Claude) directly from Java over HTTP — there is no local OCR/vision
pipeline in V1 that would need Python's ML ecosystem.

This follows [docs/tech-stack.md](../tech-stack.md)'s own stated rationale
for choosing Java for `services/core` in the first place: a mature,
strongly-typed ecosystem for auditable financial logic, which is V1's
highest-stakes code (billing, GST correctness). Splitting into two
languages for a single-service V1 would add real operational cost
(two runtimes, two dependency ecosystems, cross-language calls inside
what's supposed to be one deployable unit) for AI work that, at V1's
actual scope, doesn't yet need Python's specific strengths (local
OCR/vision models, embeddings). `services/ai` splits out as its own
Python service per [architecture.md](../architecture.md) once AI work
grows into what actually needs it (voice, embeddings, background catalog
clustering — see [ai-architecture.md](../ai-architecture.md)).

## Consequences

- `services/core` (Spring Boot, Java 21) is the only backend deployable
  unit in V1. `docs/tech-stack.md` and `docs/context.md` now say so
  explicitly, so a future session doesn't assume Python exists in V1
  because the target architecture mentions it.
- AI feature work in V1 (Module 4, Module 5, Module 10 partial) is built
  as Java code calling an external LLM API, not as calls into a local
  Python service. Prompt versioning and pipeline-stage testability
  (`docs/coding-standards.md`) still apply, just from Java.
- When AI scope grows enough to need Python (voice transcription models,
  local embeddings, background clustering per
  `docs/ai-architecture.md`), that's the trigger to split `services/ai`
  out — not a fixed calendar date, and not before.
