# Future Modules (Long-Term Vision, Not Specified)

## What this document is and isn't

This is a named list of where Nexora could go once it is a working,
validated Business Operating System for its core 12 modules
([PROJECT_SPEC.md](../PROJECT_SPEC.md)). It is **not** a spec, **not**
scheduled, and **not** something to write architecture or a PRD for until
its own trigger condition below is met. The reason this list exists as a
separate document, instead of living inside `PROJECT_SPEC.md` alongside
the active modules, is specifically to stop long-term ambition from
diluting what's actually being built right now — see
[mvp-scope.md](mvp-scope.md).

## The Zoho lesson

The nearest real comparable to "one platform that eventually does
everything a business needs" is Zoho, which now spans CRM, HR, Payroll,
Accounting, and dozens more. Zoho did not plan or architect that whole
suite up front. It shipped one product, made it profitable and genuinely
excellent, and only then built the next one — sequentially, over two
decades, funded by the previous product's success rather than outside
capital. The vision below is legitimate. The mistake would be trying to
plan or build more than one step of it at a time.

## Candidate future modules

Grouped by how directly they extend the existing core, not by importance:

**Extends the money/ledger core**
- Full accounting (general ledger, balance sheet, P&L, cash flow
  statements) — the natural extension of [gst-compliance.md](gst-compliance.md)'s
  Phase 3 filing work, and the path toward the Tally-independence
  ambition described in
  [adr/0012-tally-adoption-sequencing.md](adr/0012-tally-adoption-sequencing.md).
- Payroll, Attendance, Employee Management — a business's second-biggest
  category of recurring paperwork after billing.

**Extends operations**
- Manufacturing (BOM, production tracking) — relevant to the
  "manufacturer" segment named in `PROJECT_SPEC.md`'s target users, but a
  materially different domain model from retail/wholesale billing.
- Multi-branch management — relevant once a single pilot business
  outgrows one location, not before.
- Projects — relevant mainly to the service-business segment.

**Extends reach**
- WhatsApp Automation (beyond the input/output channel already in
  Module 4/11) — automated reminders, marketing broadcasts.
- E-commerce / marketplace integration — sell where the business already
  sells online, not just in-store.
- POS / IoT integration — physical hardware integration for larger
  retail operations.

**Extends trust and relationships**
- CRM (beyond Module 1's customer ledger) — pipeline/lead tracking for
  businesses with a real sales process, not just walk-in retail.
- AI Customer Support — support automation once there's a large enough
  user base to need it.
- Document Management, Contracts, Digital Signatures — relevant mainly
  to the service/contractor segment.

## Trigger condition for turning any of these into a real spec

A future module gets its own PRD only after the current 12 modules have
real, paying or committed-pilot customers using them — not on a fixed
calendar, and not because it sounds like a natural next feature in a
planning conversation. The [roadmap.md](roadmap.md) Phase 4/5 entries
already scheduled (predictive intelligence, autonomous agent, full
connector ecosystem) are the only forward-looking work that's actually
sequenced; everything on this page is deliberately unsequenced until
then.
