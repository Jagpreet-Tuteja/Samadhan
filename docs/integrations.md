# Integrations

Nexora's differentiator is not replacing a business's existing billing
software on day one — it is connecting to it. This document covers how
Nexora interoperates with Tally and similar systems.

## Why interoperability matters

Most target businesses already have years of data in Tally (or Busy, Marg
ERP, Zoho Books, QuickBooks, or plain Excel). Forcing a full migration
before a business can get value from Nexora is the single biggest adoption
blocker for this category of product. The Connector Framework
(`services/connectors`, see [architecture.md](architecture.md)) exists so
a business can turn on Nexora for AI billing/pricing/inventory *today*
while Tally keeps running as their accounting system of record, if they
choose.

## Tally integration

Tally exposes an XML-over-HTTP interface (Tally's built-in ODBC/XML
gateway, typically on port 9000 of the machine running Tally). The Tally
connector will:

- **Import**: pull ledgers (customers/suppliers), stock items, and
  vouchers (sales, purchase, payment, receipt) from Tally into Nexora's
  data model on initial setup and on a sync schedule.
- **Export**: push Nexora-created invoices, purchase bills, and payments
  back into Tally as vouchers, so Tally's books stay complete without
  double entry.
- **Field mapping**: a per-organization mapping between Tally ledger/stock
  groups and Nexora's categories/units, configurable during onboarding
  (`connector_configs` table, see [database.md](database.md)).
- **Conflict handling**: sync is logged append-only in
  `connector_sync_log`; conflicting edits (same voucher changed in both
  systems) are surfaced to the owner rather than auto-resolved.
- **Deployment reality**: Tally typically runs on a local Windows machine
  inside the business, not in the cloud. The connector will run as a
  small local agent/service on that machine (or on the same LAN) that
  talks to Tally's local XML interface and syncs to Nexora's cloud API
  over HTTPS — direct cloud-to-Tally XML calls are not possible when
  Tally has no public network exposure.

## Other billing/accounting software

The same connector interface (adapter implementing
import/export/field-mapping) is planned for:

- **Busy Accounting** — via Busy's data export/API.
- **Marg ERP** — via Marg's data interchange format.
- **Zoho Books / QuickBooks** — via their public REST APIs (cloud-native,
  no local agent required).
- **Generic Excel/CSV** — universal fallback import/export for supplier
  price lists, product catalogs, and customer lists, usable even where no
  dedicated connector exists yet.
- **WhatsApp Business API** — not an accounting system, but a first-class
  input/output channel: incoming price lists, voice notes, and photos via
  WhatsApp, and outgoing invoice PDFs/payment reminders.

## Risk: the local agent is a support burden, not a routine integration

Building and distributing a Windows background service that a
non-technical shop owner installs correctly, that survives antivirus
false positives, and that works across different Tally versions (Tally
Prime vs. Tally ERP 9 use different XML schemas) is a real, ongoing
support cost — closer to shipping and maintaining a small desktop product
than to calling a REST API. Treating it as a routine connector, the way
the rest of this document reads, understates that.

**Gate**: per [mvp-scope.md](mvp-scope.md), the local agent is not built
for V1. Tally-using pilot businesses are bridged with manual CSV
export/import in the meantime. Before committing engineering time to the
full agent, prototype it against 2-3 real Tally installations (ideally
spanning both major Tally versions) and confirm: a non-technical owner
can install it without support intervention, it survives common antivirus
software without triggering a false-positive block, and the sync accuracy
is high enough to trust with real ledger data. If that prototype doesn't
clear those bars, the CSV-bridge stopgap may need to remain the primary
Tally interop path for longer than originally planned, rather than
building the full agent on schedule regardless.

## Non-goals

Nexora's connectors do not attempt to replicate Tally's full GST filing
or statutory compliance workflows — Nexora reads/writes transactional
data (vouchers, ledgers, stock) for the sake of keeping both systems
consistent; compliance filing remains whichever system the business
already trusts for that, until [docs/roadmap.md](roadmap.md) Phase 3.

See [adr/0006-tally-integration.md](adr/0006-tally-integration.md) for the
accepted approach.
