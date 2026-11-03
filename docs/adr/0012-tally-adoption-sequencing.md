# 0012 — Tally Adoption Sequencing

Status: Accepted (extends [ADR 0006](0006-tally-integration.md))
Date: 2026-07-19

## Context

Two framings of Nexora's relationship to Tally have both entered this
project's planning at different points: [ADR 0006](0006-tally-integration.md)
positions Nexora explicitly as a **Tally companion** — connecting to and
syncing with Tally rather than requiring migration, because forcing a
rip-and-replace is identified in
[docs/competitor-analysis.md](../competitor-analysis.md) as the single
biggest adoption blocker in this market. A later planning pass introduced
a more aggressive framing — building toward Nexora *not depending on*
Tally and eventually making it unnecessary. These are not the same
strategy, and left unresolved, the tension leaks into real decisions:
how aggressively accounting features get built, how the product is
pitched to accountants (who are frequently the actual software-adoption
gatekeeper for a shop), and whether Tally-loyal users feel replaced or
supported.

## Decision

Both framings are true, at different times, and the sequencing matters
more than either framing alone:

1. **Now through Phase 2**: Nexora is unambiguously a companion. It syncs
   with Tally, never requires a business to abandon it, and is pitched as
   reducing daily manual work — not as a Tally alternative. This is the
   framing every pilot conversation and every piece of marketing uses.
2. **Phase 3 onward**: Nexora begins building genuinely independent
   accounting depth (general ledger, GST filing, balance sheet — see
   [gst-compliance.md](../gst-compliance.md) and
   [future-modules.md](../future-modules.md)) because those are valuable
   in their own right — reducing a business's dependence on manual
   accountant coordination — not because the goal is explicitly to
   eliminate Tally.
3. **Tally independence, if it happens, is an outcome a business chooses
   for itself** once Nexora's own accounting depth is good enough to earn
   that trust — it is never the product's stated near-term objective, and
   is never used as a pitch against Tally. The distinction matters: "we
   make your daily work easier" keeps accountants and Tally-loyal owners
   as allies; "we make Tally unnecessary" makes them adversaries before
   Nexora has earned the right to that claim.

## Consequences

- Marketing, sales conversations, and onboarding language must stay in
  companion framing through Phase 2, even as Phase 3+ engineering quietly
  builds toward independence — this needs to be an explicit discipline,
  not an assumption, since it's easy for an ambitious team to drift into
  "we're replacing Tally" language once independent accounting features
  exist.
- Full accounting depth is real, scheduled work (Phase 3+, tracked in
  [future-modules.md](../future-modules.md)) — this ADR does not deprioritize
  it, it only fixes how it's framed and sequenced relative to the
  existing companion strategy.
- Revisit this sequencing if pilot data (per
  [mvp-scope.md](../mvp-scope.md)'s validation gate) shows businesses
  actively want a faster path away from Tally — that would be a reason to
  accelerate Phase 3 accounting work, not a reason to change the
  companion-first framing before then.
