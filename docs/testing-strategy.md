# Testing Strategy

Status: planned, no test suite exists yet.

## Layers

- **Unit tests** — domain logic in `services/core` (pricing, tax
  calculation, inventory ledger math) and pure extraction/parsing
  functions in `services/ai`.
- **GST correctness tests** — a dedicated fixture suite (intrastate,
  interstate, composition scheme, reverse charge) checked against
  known-correct tax breakdowns, held to a 100%-correct bar since this is
  deterministic business logic, not an AI accuracy problem — see
  [gst-compliance.md](gst-compliance.md). Runs separately from the AI
  extraction-accuracy fixtures below.
- **Integration tests** — API endpoints against a real test database;
  connector adapters against recorded fixtures (e.g. a captured Tally XML
  response) rather than a live Tally instance in CI.
- **Tenant isolation tests** — automated checks that cross-tenant access
  is impossible at both the application-layer scoping and the database
  RLS layer (see [adr/0013-row-level-security.md](adr/0013-row-level-security.md)),
  not just one or the other.
- **Offline sync tests** — once [offline-strategy.md](offline-strategy.md)
  ships in Phase 2, conflict scenarios (two devices editing related data
  while disconnected) get their own fixture suite; not needed for the V1
  online-only web app.
- **End-to-end tests** — critical owner flows (create invoice via text,
  approve a price-update batch, view dashboard) driven through the actual
  UI.

## AI accuracy evaluation

Because Module 4/5/6/10 correctness is measured in *extraction accuracy*,
not just pass/fail:

- Maintain a growing fixture set of real-world-shaped inputs (sample
  supplier PDFs, sample handwritten notes, sample voice transcripts) with
  hand-labeled expected structured output.
- Run the AI pipeline against the fixture set on every change to
  `services/ai` and track precision/recall per field (product match,
  quantity, price) over time.
- Target from [PROJECT_SPEC.md](../PROJECT_SPEC.md): less than 1% manual
  correction after AI extraction — track actual owner-correction rate in
  production (via `ai_extraction_jobs`, see [database.md](database.md))
  as the real-world metric, not just the offline fixture score.

## Catalog quality metrics

Product identity resolution (Module 12,
[product-catalog-lifecycle.md](product-catalog-lifecycle.md)) has its own
failure modes beyond field-level extraction accuracy — a wrong decision
here either pollutes the catalog with duplicates or incorrectly merges
two real products. Track, per organization and in aggregate:

- **Catalog completeness** — share of products with `pricing_status =
  priced` vs. `price_missing`.
- **Duplicate rate** — estimated share of the catalog that is an
  undetected duplicate, sampled periodically via the same embedding
  clustering used for the background sweep.
- **Merge-suggestion acceptance rate** — of high-confidence merge
  prompts shown, what fraction the owner accepts vs. rejects; a low
  acceptance rate signals the confidence thresholds need retuning.
- **Auto-link correction rate** — of near-certain auto-linked aliases,
  what fraction the owner later undoes via the weekly digest; this is the
  metric that governs whether the near-certain tier's threshold is set
  correctly, since those merges are never shown for confirmation before
  they happen.

## Confidence threshold calibration

The identity-resolution confidence tiers in
[product-catalog-lifecycle.md](product-catalog-lifecycle.md) (99%/80%
band boundaries) are documented as uncalibrated starting defaults, not
measured values — there is no real catalog to have measured them against
yet. Calibrating them against real embedding output on real pilot product
names is a required step of the [mvp-scope.md](mvp-scope.md) validation
gate, not an optional tuning pass after launch.

## CI gate

Every PR: lint, typecheck, unit tests, integration tests. AI fixture
accuracy regression (a drop below the current baseline) blocks merge for
`services/ai` changes.
