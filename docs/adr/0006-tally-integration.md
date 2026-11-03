# 0006 — Tally Integration

Status: Accepted, extended by [ADR 0012](0012-tally-adoption-sequencing.md)
Date: 2026-07-19

> ADR 0012 makes explicit how the companion positioning below sequences
> against Nexora's longer-term accounting ambition ([future-modules.md](../future-modules.md)) —
> read it alongside this ADR, not as a replacement for it.

## Context

Most target businesses already run Tally, typically on a local Windows
machine with no public network exposure. Requiring migration away from
Tally before a business can benefit from Nexora is a major adoption
blocker (see [docs/competitor-analysis.md](../competitor-analysis.md)).
Tally exposes an XML-over-HTTP interface locally (commonly port 9000) but
is not reachable directly from the public internet in a typical SMB
setup.

## Decision

Build the Tally connector as a small local agent/service that runs on the
same machine or LAN as Tally, talks to Tally's local XML interface, and
syncs to Nexora's cloud API over HTTPS. Two-way sync: import
ledgers/stock/vouchers on setup, export Nexora-created
invoices/bills/payments back into Tally as vouchers. See
[docs/integrations.md](../integrations.md).

## Consequences

- Requires building and distributing a lightweight local agent (likely a
  small Windows service/executable), which is new deployment surface
  area beyond the pure-cloud services.
- Sync conflicts (same voucher edited in both systems) are surfaced to
  the owner, not auto-resolved — logged append-only in
  `connector_sync_log`.
- Other cloud-native accounting systems (Zoho Books, QuickBooks) do not
  need this local-agent pattern and integrate via direct API calls
  instead.
