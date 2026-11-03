# 0009 — Product Catalog Identity Resolution

Status: Accepted
Date: 2026-07-19

## Context

Nexora's target businesses do not start with a clean, complete product
catalog (see [docs/product-catalog-lifecycle.md](../product-catalog-lifecycle.md)).
The catalog must be built incrementally from supplier bills, sales
invoices, photos, voice, and handwritten notes. This creates a hard
identity problem: the same physical product is routinely described
differently across sources ("Panel", "PVC Panel", "PVC White Panel 8ft",
"8ft White PVC Panel" from a different supplier). A naive design that lets
AI extraction write directly into the canonical `products` table on every
mention will silently fill the catalog with duplicates, and a design that
requires the owner to manually dedupe after the fact will not be used.

## Decision

1. AI extraction never writes directly into the canonical `products`
   table. Every product mention from any source first becomes a row in a
   staging table, `candidate_products`.
2. A candidate is resolved against the existing catalog using a combined
   score: normalized fuzzy text match, structured attribute extraction
   (size/color/brand/material/pack size), category/supplier context, and
   embedding-based semantic similarity.
3. Resolution outcome is tiered by confidence, not a single threshold:
   near-certain matches auto-link as an alias (logged for later
   spot-check); high-confidence matches prompt an explicit merge
   decision; low-confidence non-matches become new products immediately
   but are queued into a periodic background clustering job that compares
   the whole catalog, not just one new candidate against it.
4. A merge decision the owner rejects is recorded permanently in
   `confirmed_distinct_pairs` and must never be re-suggested for that
   exact pair.
5. Confirmed alternate names are retained as `aliases` on the canonical
   product rather than discarded, so matching accuracy compounds over
   time for that organization.

See [docs/database.md](../database.md) for the resulting schema and
[docs/ai-architecture.md](../ai-architecture.md) for the pipeline stage
this decision extends.

## Consequences

- Adds a staging/review layer (`candidate_products`, the Product Approval
  Queue) between extraction and commit for products specifically, on top
  of the general draft-confirmation flow already used for invoices and
  price updates (see [ADR 0005](0005-ai-integration-architecture.md)).
- Requires a background clustering job (embeddings over the full catalog)
  in addition to live, per-request matching — real infrastructure, not
  just a prompt change, and should be scoped as its own implementation
  task rather than assumed to fall out of Module 4/5 work.
- Prevents the specific failure mode that makes most billing-software
  catalogs unusable at scale (undetected duplicates accumulating silently
  over months), which this project treats as more damaging to trust than
  asking one extra confirmation question.
- Per-organization learning only, by default — no cross-tenant data
  sharing. A future opt-in cross-tenant alias dictionary to help brand-new
  organizations cold-start faster is deliberately deferred; see the
  Phase 4/5 note in [docs/roadmap.md](../roadmap.md) and the privacy
  caveat in [docs/security.md](../security.md). Nothing in this decision
  depends on that future feature existing.
