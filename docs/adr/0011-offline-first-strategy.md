# 0011 — Offline Strategy

Status: Accepted
Date: 2026-07-19

## Context

The original architecture assumed always-on connectivity with no
explicit decision made about it — a silent gap, not a considered
tradeoff. The target market (hardware, medical, grocery, wholesale shops)
routinely experiences unreliable connectivity, and the closest
incumbent, Tally, is desktop software specifically because of this
reality. Shipping true offline-first (local database + sync engine) as
part of the initial web MVP would meaningfully slow down validating the
core AI-billing loop, which is the thing most in need of fast validation
right now (see [mvp-scope.md](../mvp-scope.md)).

## Decision

V1 (web app) requires connectivity to bill — an explicit, temporary
tradeoff, not a permanent design. The API and data model are built now
with idempotency keys, client-generated UUIDs, and an explicit
conflict-resolution rule (server timestamp authoritative for stock
ordering; invoice numbers reserved server-side) so that offline support
is additive later, not a rearchitecture. True offline-first ships with
`apps/mobile` in Phase 2. See
[docs/offline-strategy.md](../offline-strategy.md).

## Consequences

- V1 pilot businesses with genuinely unreliable connectivity may have a
  degraded experience at the exact moment billing matters most — this is
  a real, accepted cost of moving fast, and the MVP validation gate
  (see [mvp-scope.md](../mvp-scope.md)) explicitly measures how often
  this actually happens, so the tradeoff gets revisited with data rather
  than assumption.
- Building the sync-safe API contract into V1 (idempotency keys,
  client-generated UUIDs) costs a small amount of extra V1 engineering
  effort up front, in exchange for not having to redesign the API when
  offline support is added in Phase 2.
- AI-driven modules (billing extraction, price intelligence) remain
  connectivity-dependent by nature regardless of this decision — input
  capture can be queued offline, but structuring cannot happen offline.
