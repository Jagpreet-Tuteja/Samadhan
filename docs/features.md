# Feature Register

Status legend: `planned` (designed, not built), `in-progress`, `shipped`.

| Module | Feature | Status |
|---|---|---|
| Foundation | Monorepo skeleton, tenancy/identity schema, Row-Level Security | shipped |
| Authentication | Phone+OTP, JWT access/refresh rotation, signup, multi-org selection/switching | shipped |
| 1. Customer Management | Customer profiles, ledger, search | shipped |
| 2. Inventory | Products, categories, auto-SKU, single warehouse, stock ledger, low-stock (V1 scope) | shipped |
| 2. Inventory | Unit conversions, multi-warehouse, batch/expiry, barcode scanning | planned |
| 3. Billing | GST tax invoice and bill of supply: CGST/SGST/IGST, composition, place of supply, idempotent issue, stock + ledger effects (V1 scope) | shipped |
| 3. Billing | Invoice PDF, quotation, purchase bill, credit/debit note, delivery challan | planned |
| 4. AI Billing | Text/voice/photo/PDF/WhatsApp → invoice | planned |
| 5. Price Intelligence | Manual retail price + append-only price history (the tables Module 5 writes to) | shipped |
| 5. Price Intelligence | Price ingestion, diff, approval, batch undo | planned |
| 6. Inventory Intelligence | Fast/slow movers, reorder suggestions | planned |
| 7. Supplier Management | Supplier profiles, purchase history, auto-PO | planned |
| 8. Expense Management | Expense entry, categorization, analytics | planned |
| 9. Business Dashboard | Sales/profit/cash flow summary, AI summary | planned |
| 10. AI Business Assistant | Conversational query interface | planned |
| 11. Connector Framework | Tally, Busy, Marg, Zoho, QuickBooks, CSV | planned |
| 12. Catalog Intelligence | Auto-SKU, `price_missing` state | shipped |
| 12. Catalog Intelligence | Candidate staging, identity resolution, aliases, approval queue | planned |

Module 1 shipped 2026-09-13; Module 2's V1 scope (with manual pricing,
and the auto-SKU/`price_missing` pieces of Module 12) shipped
2026-11-03, as did Module 3's V1 scope (GST invoices). The rest are still
`planned`.
See [docs/progress.md](progress.md). Update each row's status as
implementation proceeds, and add sub-rows for major feature slices as they
get built.
