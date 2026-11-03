# Offline Strategy

## The problem this fixes

The original architecture was cloud-only with no offline design at all —
a silent assumption, not a decision. That's a real mismatch: the target
businesses (hardware stores, medical stores, wholesalers) are exactly the
kind of shop where a flaky connection or a power cut is routine, and
**Tally itself is a desktop app specifically because shops need to keep
working without internet.** An invoicing app that can't create an invoice
because the WiFi dropped fails at the one moment it matters most — a
customer standing at the counter.

## V1 decision (explicit, not silent)

Per [mvp-scope.md](mvp-scope.md), **V1 is a web app and requires
connectivity to bill.** This is a deliberate, temporary tradeoff to
validate the core loop quickly, not a permanent architectural stance —
see [adr/0011-offline-first-strategy.md](adr/0011-offline-first-strategy.md).
The validation gate explicitly requires checking how often pilot
businesses actually lose connectivity mid-billing, which determines how
urgently this needs to change.

## What V1 must still get right, so offline can be added later without a rewrite

Even though V1 doesn't ship offline support, the API and data model are
designed now to make adding it later additive, not a rearchitecture:

- **Client-generated idempotency keys** on every mutating request
  (invoice creation, payment recording, stock adjustment) — so a client
  that retries a request after a dropped connection never creates a
  duplicate.
- **Client-generated UUIDs** for invoices and other created entities
  rather than server-assigned sequential IDs — a client can safely create
  a record locally before it has ever talked to the server.
- **A conflict resolution rule decided now, not discovered during an
  incident**: for stock ledger ordering, the server's received timestamp
  is authoritative, not the client's clock (clients can't be trusted to
  agree on wall-clock time); for human-facing invoice numbering, numbers
  are reserved through a server call, not generated offline, to avoid two
  devices independently minting the same invoice number.

## Phase 2: true offline-first, on mobile

[docs/roadmap.md](roadmap.md) Phase 2 ships `apps/mobile`. This is where
offline-first actually gets built properly: a local on-device database,
writes that succeed locally first and sync in the background, and a
sync-conflict UI for the rare case two devices genuinely disagree. Mobile
gets this treatment (not the V1 web app) because the phone in a shop
owner's hand is the realistic primary device for day-to-day billing at
scale, and is where offline reliability matters most.

## What never works offline, by nature, and how that's handled

OCR, LLM structuring, and voice transcription (Module 4/5) require a
live AI service call — there is no offline version of "understand this
photo." The offline-tolerant design here is: the raw input (photo, voice
recording, typed text) can always be captured and queued locally even
with no connection, and is processed automatically the moment
connectivity returns, rather than blocking the owner from capturing it at
all. The owner is never told "you can't do that right now" for the act of
capturing input — only structured-data confirmation waits for
connectivity.
