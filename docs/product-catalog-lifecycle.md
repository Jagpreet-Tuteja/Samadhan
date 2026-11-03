# Product Catalog Lifecycle (Self-Growing Catalog)

Covers Module 12 (Catalog Intelligence) from [PROJECT_SPEC.md](../PROJECT_SPEC.md).

## The problem

A new business has no catalog. A real business does not have 300 products
sitting in a clean spreadsheet, waiting to be imported — it has a mix of
memory, handwritten lists, old supplier PDFs, and paper bills, and even
the owner does not always know today's price for every product.

The naive design assumes the catalog is a precondition: "the user enters
300 products, then the software becomes useful." That assumption is wrong
for this market and is why existing billing apps feel like a chore to set
up. Nexora's catalog must **grow through normal use**, not through an
onboarding form.

## Core principle

Every interaction that mentions a product — a supplier bill, a sales
invoice, a purchase order, a photo, a voice note, an Excel sheet, a PDF —
is a chance to add to or refine the catalog. The owner should almost never
feel like they are "entering products"; they are just running their
business, and the catalog fills itself in behind the scenes.

The one hard rule that makes this safe (see
[ai-architecture.md](ai-architecture.md) and
[adr/0009-product-catalog-identity-resolution.md](adr/0009-product-catalog-identity-resolution.md)):
**AI extraction never writes directly into the canonical `products`
table.** Every product mention first becomes a row in `candidate_products`
(see [database.md](database.md)). Only a confirmed, matched, or
owner-approved candidate ever becomes — or updates — a real product. This
is what prevents the catalog from silently filling up with duplicates and
OCR misreads.

## Onboarding: Scenario 1 — brand-new business, zero products

On first login, product count is 0. Nexora does not show a blank "add
product" form and stop there. It offers every ingestion path at once:

```
How would you like to set up your business?

📷 Scan a supplier catalog or rate list
📄 Upload an Excel sheet
📑 Upload a PDF
📸 Photograph a handwritten product list
⌨️  Add products manually
🔄 Import from Tally (see integrations.md)
🚀 Start empty — I'll add products as I go
```

"Start empty" is a first-class, fully supported option — not a fallback.
The rest of this document is what happens after that choice, because most
real businesses effectively choose it whether they mean to or not: they
start selling, and the catalog builds itself from there.

## Scenario 2 — handwritten list, no prices

Owner photographs:

```
PVC Panel White
PVC Panel Brown
Corner
Adhesive
PVC Ceiling
```

The AI pipeline (OCR → LLM structuring, see
[ai-architecture.md](ai-architecture.md)) extracts each line as a
candidate product with whatever it can infer and leaves the rest blank
rather than guessing:

| Field | Value |
|---|---|
| Name | PVC Panel White |
| Category | PVC Panels (inferred from name pattern) |
| Unit | Piece (inferred from category default) |
| GST rate | *(ask user)* |
| Price | *(unknown)* |

Nexora only asks for what is actually missing and actually needed to
proceed — never re-asks for something it could infer with high
confidence, and never blocks the owner on fields that aren't required yet
(see "price-missing is not an error state" below).

## Scenario 3 — products entered without a price

If price truly isn't known yet, the product is still created — with
`pricing_status = price_missing` (see [database.md](database.md)) — so it
exists in the catalog, is searchable, and can be matched against later,
even before it has a price:

| Product | Status |
|---|---|
| PVC Panel White | Price Missing |
| PVC Panel Brown | Price Missing |

`price_missing` products surface as a small, dismissible dashboard nudge
("2 products are missing a price"), never as a blocking error — the owner
can sell, quote, or otherwise reference the product; only invoicing it
requires a price to be supplied, at which point the owner is prompted
once, inline, rather than being sent back to a separate screen.

## Scenario 4 — a supplier invoice creates products automatically ⭐

This is the highest-leverage flow in the whole system, because it asks
the owner for nothing beyond what they were already going to do (upload
the supplier's bill).

Owner uploads a supplier PDF. AI extraction (see
[ai-architecture.md](ai-architecture.md)) detects:

```
PVC Panel White   ₹650
PVC Corner        ₹120
Glue              ₹280
```

`PVC Panel White` and `PVC Corner` match existing products (see identity
resolution below) and simply update their supplier cost / trigger a price
diff (see [pricing-engine.md](pricing-engine.md)). `Glue` matches nothing
in the catalog, so it becomes a new `candidate_products` row and is
surfaced once, batched with any other unmatched lines from the same
document:

```
1 product from this bill doesn't exist yet:

  Glue — ₹280 (from ABC Industries)

  [Add to catalog]   [Skip]
```

On "Add to catalog," a real `products` row is created, seeded with name,
supplier, and price from the bill, `pricing_status = priced`. The catalog
grew by one product without the owner ever opening a "products" screen.

## Scenario 5 — the first sale creates a product

Owner types (or says): `10 PVC Panels`. No match is found. Rather than
silently failing or silently guessing, the AI asks the minimum necessary:

```
"PVC Panels" doesn't match anything in your catalog yet.
Is this a new product?               [Yes] [No, let me search again]

Selling price?  ______
```

Owner answers `650`. The product is created and the original invoice line
now resolves against it — saved permanently, and available for every
future match.

## Scenario 6 — near-duplicate names over time (the alias problem)

Owner enters `Panel` on day one, `PVC Panel` on day two, `PVC Panel White`
on day three. Naively, this creates three products. Instead, each new
mention runs through identity resolution against the existing catalog; if
similarity is high enough, Nexora asks rather than assumes:

```
"PVC Panel White" looks like it might be the same product as
"PVC Panel" you already have. Merge them?

  [Yes, same product]   [No, these are different]
```

**If the owner says "No"**, the pair is recorded in
`confirmed_distinct_pairs` (see [database.md](database.md)) so Nexora
never asks about that exact pair again — an easy detail to miss, and the
difference between a helpful assistant and an annoying one.

**If the owner says "Yes"**, the two names become aliases of one
canonical product (see "Canonical product + aliases" below) — no history
is lost; every past invoice that used either name still resolves
correctly.

## Scenario 7 — same product, different supplier naming

Supplier A writes `PVC Panel White 8 Feet`; Supplier B writes `8ft White
PVC Panel`. Token order is different, abbreviations differ, but the
underlying product — and its extracted attributes (type: PVC Panel,
color: White, size: 8ft) — match. Identity resolution scores this as a
high-confidence possible match and prompts a merge, the same as Scenario
6. This is fundamentally a fuzzy text + structured attribute matching
problem, not a plain string-equality problem — see "How matching actually
works" below.

## Scenario 8 — no SKUs

Most local businesses have never used a SKU system. Nexora auto-generates
one at creation time so every product still has a stable machine
identifier for barcode/QR printing and integrations, without ever asking
the owner to invent one:

```
PVC Panel White   → PVC-0001
PVC Corner        → PVC-0002
Glue              → GLU-0001
```

See [database.md](database.md) for the generation scheme.

## Scenario 9 — bulk price updates at scale

A catalog of 5,000 products receives a new 100-line supplier bill. Most
lines are unchanged. The diff (see [pricing-engine.md](pricing-engine.md))
surfaces only the 12 that actually changed:

```
12 of 100 products changed price.

  [Review one by one]   [Update All]
```

Any of the 100 lines that don't match an existing product route into the
same candidate-product flow as Scenario 4, alongside the price diff —
catalog growth and price intelligence share one ingestion pipeline, not
two separate ones.

## Scenario 10 — a genuinely new, more specific product

Owner writes `Special Blue Panel Imported`. No match, and not similar
enough to anything existing to suggest a merge. Nexora asks only for what
it cannot infer, using the organization's [industry
configuration](industry-config.md) to shape the question (a hardware
store gets asked about size/material; a grocery store would be asked
about weight/pack size instead):

```
New product: Special Blue Panel Imported

  Category?  PVC Panel  (suggested)
  GST rate?  18%         (suggested, matches category default)
  Unit?      Piece       (suggested, matches category default)
  Price?     ______
  Supplier?  ABC Industries (suggested, matches upload source)
```

Suggested values are pre-filled and editable, not blank fields — the
owner is confirming, not typing from scratch.

## How matching actually works

Text similarity alone is not enough — "Panel" and "PVC Panel" and "PVC
White Panel 8ft" need more than string distance to resolve correctly.
Matching combines five signals:

1. **Normalized fuzzy text match** — case/spacing/punctuation-insensitive
   token comparison (e.g. trigram similarity or edit distance on
   normalized tokens).
2. **Structured attribute extraction** — size, color, brand, material,
   and pack size are pulled out of the raw text into explicit fields
   (`attributes` JSON on the candidate/product) both for matching *and*
   for asking a precise clarifying question instead of a vague one
   ("Color — White or Brown?" instead of "is this the same product?").
3. **Category and supplier context** — a candidate from the same supplier
   as an existing product, in the same inferred category, raises
   confidence; a candidate from an unrelated category lowers it even if
   the text is similar.
4. **Price consistency** — a large, unexplained price gap between a
   candidate and its best text match *lowers* confidence rather than
   being ignored. Two lines both called "Cement Bag" at ₹380 and ₹650 are
   more likely a weight/brand variant or a typo than the same product;
   treating price as evidence, not just an output of the match, avoids
   proposing merges that look textually obvious but are financially
   implausible. A large price gap doesn't block a merge outright (the
   same product's price can legitimately jump), but it pushes a candidate
   that would otherwise be near-certain down into the high-confidence
   tier, so it gets a prompt instead of a silent auto-link.
5. **Semantic similarity (embeddings)** — catches cases plain fuzzy text
   matching misses, such as synonyms or reordered descriptive phrases,
   and is what powers the periodic background clustering pass described
   below.

### Confidence tiers

| Tier | Approx. range | Behavior |
|---|---|---|
| Near-certain | ≥99%, same tokens modulo case/spacing/attributes, no material price gap | Auto-linked as an alias, surfaced immediately as a dismissible inline confirmation ("Linked 'PVC Panel White' to 'Panel' — Undo?") — see "Why near-certain still isn't silent" below. |
| High confidence | Roughly 80–98%, or a near-certain text match with a material price gap | Interrupts with an explicit merge prompt (Scenarios 6 and 7), one tap to confirm or reject. |
| Low confidence | Below ~80% | Treated as a genuinely new product immediately (never blocks the owner's current action) but queued into a periodic background "possible duplicates" sweep — live, single-item matching during one extraction will miss pairs that only become obvious once the whole catalog is compared at once. |

**These band boundaries (99%, 80%) are starting points, not measured
values.** No real embedding model has been run against this project's
actual product-name pairs yet — there is no catalog to run it against.
Treat them as configurable defaults to be recalibrated against real
fixture and pilot data before launch, per
[testing-strategy.md](testing-strategy.md) and the MVP validation gate in
[mvp-scope.md](mvp-scope.md), not as validated engineering constants.

### Why near-certain still isn't silent

[PRODUCT_PRINCIPLES.md](../PRODUCT_PRINCIPLES.md) states plainly: "AI
proposes; the owner approves anything that touches money, price, or
stock. No silent financial writes." Merging two products' identities
combines their stock ledgers — that is a real inventory change, so a
fully silent auto-link (originally described here as logged only to a
*weekly* digest) directly violated that rule. The fix is not to remove
the near-certain tier's speed — a merge this obvious shouldn't cost the
owner a decision — but to make the confirmation cheap instead of absent:
a single-line, immediately visible, one-tap-to-undo inline notice at the
moment it happens, not a delayed digest the owner might never open. Fast
and confirmed are not in tension here; silent and confirmed are.

### Background clustering: how it actually runs

The low-confidence tier's periodic sweep is real infrastructure, not a
detail to wave at:

- Runs on a schedule proportional to catalog size — nightly for catalogs
  above roughly 500 products, on-demand (triggered after a bulk import)
  below that, rather than a fixed interval regardless of scale.
- Uses an approximate nearest-neighbor index over product embeddings
  (e.g. pgvector with HNSW indexing) rather than full pairwise comparison
  across the catalog — pairwise comparison is quadratic and stops being
  affordable well before a catalog reaches the sizes this system is
  designed for.
- Is scoped as its own implementation task when Module 12 gets built —
  see [adr/0009-product-catalog-identity-resolution.md](adr/0009-product-catalog-identity-resolution.md) —
  not something that falls out of the live per-request matching code for
  free.

## Canonical product + aliases

Once two names are confirmed as the same product, both remain searchable:
the canonical product record stores an `aliases` list (every name it has
ever been called, across every input source). This is a compounding
asset — every confirmed alias makes every *future* OCR read, voice
transcription, or typed mention of that product match faster and more
confidently, for that organization. The catalog does not just grow; it
gets more accurate at recognizing itself over time.

## The AI Product Approval Queue

Every AI-detected product that isn't an existing, confidently-matched
product lands in a review queue instead of being silently created or
silently discarded:

```
🔔 8 new products detected from today's uploads

  ✓ PVC Panel White 8ft
  ✓ PVC Corner Premium
  ✓ Adhesive 500ml
  ✓ Ceiling Strip
  ...

  [Approve All]   [Review One by One]   [Reject]
```

This keeps onboarding fast (bulk approve when extraction is clearly
right) while keeping the catalog clean (individual review when it isn't).
The same queue surfaces high-confidence merge suggestions (Scenario 6/7)
and a running history of near-certain auto-links (each individually
undoable from the moment it happens, per "Why near-certain still isn't
silent" above), so there is one place the owner checks, not several.

## Cold start across organizations (future idea, not built by default)

Every organization currently learns its catalog and aliases independently
— a brand-new tenant starts from zero pattern knowledge even though
"PVC Panel" means the same thing to every hardware store in the country.
A future, opt-in, anonymized cross-tenant alias/category dictionary could
let a new organization's catalog bootstrap faster by reusing patterns
learned across other businesses in the same industry, without exposing
any tenant's actual product list, pricing, or business data to another
tenant. This is explicitly a Phase 4/5 idea (see
[roadmap.md](roadmap.md)) and requires a privacy design pass before it is
built — see the note in [security.md](security.md). It is not assumed or
relied on anywhere else in this document; every scenario above works
correctly for a single organization with no shared data at all.

## Why this is the actual product moat

Any billing app can render an invoice form. Very few make the first 90
days of catalog-building effortless, and fewer still keep a large catalog
clean as it grows past the point where a human would notice duplicates by
eye. Getting product identity resolution right — never silently creating
duplicates, never nagging about already-rejected merges, always asking
the minimum necessary question — is what makes Nexora feel like it
"already knows" a business instead of feeling like a database the owner
has to maintain.
