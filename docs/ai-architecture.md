# AI Architecture

## Goal

Turn unstructured input — typed text, voice, a photo of handwritten
stock, a supplier's PDF price list, a WhatsApp message, a scanned business
card — into structured, validated business data, with the owner
confirming anything that touches money, price, or stock.

## Pipeline stages

1. **Input capture** — text, voice (audio), image, PDF, or forwarded
   WhatsApp/email content, tagged with source and organization.
2. **Input classification** — determine intent (create invoice, update
   price, add customer, ask a question) and modality (needs OCR? needs
   speech-to-text?).
3. **Modality-specific extraction**
   - Speech-to-text for voice notes.
   - OCR + layout analysis for photos/PDFs of bills, handwritten notes,
     supplier catalogs, and business cards.
   - Direct parsing for structured input (CSV/Excel supplier lists).
4. **LLM structuring** — the extracted text/data is passed to an LLM with
   the organization's product/customer catalog as grounding context, and
   turned into a typed draft object (e.g. `InvoiceDraft`,
   `PriceUpdateBatch`, `CustomerDraft`) with a confidence score per field.
5. **Entity resolution** — match extracted names against existing
   customers/suppliers/products in `services/core`; when more than one
   plausible match exists (or none), mark `needs_clarification` instead
   of guessing. For products specifically, this stage is the product
   identity resolution pipeline described below — the highest-stakes
   part of entity resolution, because getting it wrong either creates a
   duplicate product or silently merges two different ones.
6. **Owner confirmation** — the draft (and, for price updates,
   specifically only the *changed* fields — old value, new value, margin
   impact) is shown to the owner. Nothing commits without confirmation.
7. **Commit** — on confirmation, `services/ai` calls the relevant
   `services/core` endpoint; `services/core` performs the actual
   transactional write.

## Product identity resolution

Every product mention extracted anywhere in the pipeline — from an
invoice line, a supplier PDF, a voice note, a photo — is written to
`candidate_products` first, never directly to `products` (see
[adr/0009-product-catalog-identity-resolution.md](adr/0009-product-catalog-identity-resolution.md)).
Resolving a candidate against the existing catalog combines five signals:

1. Normalized fuzzy text match on product name and known aliases.
2. Structured attribute extraction (size, color, brand, material, pack
   size) pulled into an `attributes` JSON, compared field-by-field.
3. Category and supplier context from the current extraction (a
   candidate from the same supplier/category as an existing product
   raises confidence).
4. Price consistency — a large, unexplained price gap between a
   candidate and its best text match lowers confidence rather than being
   ignored, since a near-identical name at a very different price is
   often a different SKU, not the same product.
5. Embedding-based semantic similarity, which also powers a periodic
   background job that re-clusters the *entire* catalog on a
   scale-proportional schedule using an approximate nearest-neighbor
   index (e.g. pgvector/HNSW), not full pairwise comparison — live
   one-candidate-at-a-time matching during a single extraction will miss
   pairs that only become obvious when compared against everything else.

The combined score sorts into three tiers — near-certain (auto-link as
alias, surfaced immediately as an undoable inline confirmation, never
fully silent, per [PRODUCT_PRINCIPLES.md](../PRODUCT_PRINCIPLES.md)'s "no
silent financial writes" rule), high-confidence (interrupt with an
explicit merge prompt), and low-confidence (create as new immediately,
queue for the background clustering pass). The tier boundaries are
starting defaults to be recalibrated against real data, not measured
constants — see [testing-strategy.md](testing-strategy.md). See
[product-catalog-lifecycle.md](product-catalog-lifecycle.md) for the full
walkthrough with examples, and [database.md](database.md) for
`candidate_products`, `product_sku_sequences`, and
`confirmed_distinct_pairs`.

A merge the owner explicitly rejects is written to
`confirmed_distinct_pairs` and permanently excluded from future
suggestions — the identity resolution step checks this table before
proposing any merge, so a rejected pair is never re-surfaced.

## Confidence and trust

- Every extracted field carries a confidence score.
- Below a configurable threshold, the field is flagged for manual review
  rather than silently accepted.
- Every AI-driven change is logged in `ai_extraction_jobs` with the raw
  input, the draft, the final confirmed values, and who confirmed —
  giving a full audit trail and a dataset for measuring extraction
  accuracy over time (see [testing-strategy.md](testing-strategy.md)).

## Model strategy

- Start with hosted LLM APIs (e.g. Claude) for language understanding and
  structuring — no need to train a model to ship the MVP.
- Use a dedicated OCR/vision engine for document/image extraction,
  fed into the LLM structuring step rather than asking a single model to
  do both raw pixel-level OCR and business reasoning.
- Speech-to-text and text-to-speech via a hosted API for voice input and
  spoken responses.
- Keep the LLM prompt/response contract typed and versioned in
  `services/ai`, so prompts can be iterated without touching
  `services/core`.
- Revisit self-hosted/fine-tuned models only once volume and unit
  economics justify it (see [monetization.md](monetization.md)) — not a
  Phase 1 concern.

## Why AI is a separate service

- `services/core` must stay deterministic, auditable, and independently
  testable — it should never depend on an LLM being available or
  correct.
- AI providers/models will change faster than the transactional data
  model; isolating them avoids coupling the two lifecycles.
- Different AI workloads (OCR batch jobs, live chat, voice) have very
  different scaling and latency profiles from the core API.

See [adr/0005-ai-integration-architecture.md](adr/0005-ai-integration-architecture.md).
