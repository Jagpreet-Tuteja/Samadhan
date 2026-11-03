# Nexora Project Specification

Version: 1.0
Status: Early implementation — see [docs/context.md](docs/context.md)
for exactly where things stand.

## Project

Name: Nexora

Working tagline: **The AI Business Operating System**

Mission: Build an AI-powered business operating system that becomes the
digital brain of every small and medium business — understanding natural
language, images, voice, and documents to automate the repetitive
administrative work that currently eats hours of every business owner's
day.

## Problem

Millions of small and medium businesses waste hours every day on repetitive
administrative work:

- Creating invoices manually.
- Updating thousands of product prices.
- Managing inventory.
- Tracking customers.
- Managing suppliers.
- Recording expenses.
- Searching old bills.
- Calculating GST.
- Handling quotations.
- Creating purchase orders.

Most businesses run this on a stack of disconnected tools — Tally, Excel,
WhatsApp, a calculator, a notebook, paper bills — which causes human
errors, time loss, incorrect pricing, stock mismatches, lost customers, and
poor business analytics.

## Product Philosophy

Nexora is not another accounting app, not another billing app, not another
ERP clone, and not a Tally replacement. It is a conversational, AI-native
layer that sits on top of a business's data — and, where a business already
depends on Tally or similar software, it **connects to and augments** that
system rather than forcing a rip-and-replace migration.

The user should never feel they are operating software. It should feel
like talking to an intelligent employee:

```
Owner:  Create invoice for Rahul.
Nexora: Which Rahul?
Owner:  Rahul Traders.
Nexora: Found Rahul Traders. Previous balance: ₹12,300. Create today's invoice?
Owner:  Yes. 10 PVC Panels, 5 Adhesives, 2 Corners.
Nexora: Invoice created. Inventory updated. Customer history updated.
        Profit calculated. PDF generated. WhatsApp ready.
```

All in under 20 seconds, end to end.

## Target Users

- Hardware stores
- Electrical shops
- Medical stores
- Grocery stores
- Furniture dealers
- Clothing stores
- Wholesalers
- Manufacturers
- Service businesses
- Restaurants
- Construction suppliers
- Automobile parts dealers
- Any retail or wholesale business currently on Tally, Excel, or paper

## Long-Term Vision

One platform. Every business. Not software built for a single industry —
a system that adapts to any industry through configurable products,
pricing, taxes, units, and workflows instead of hardcoded industry logic.

## Core Modules

### Module 1 — Customer Management
Customer profiles (address, GST number, phone, purchase history, pending
balance, credit limit, favourite products, notes). Search by phone, name,
GST, previous invoice, voice, or AI chat.

### Module 2 — Inventory
Unlimited products, categories, brands, barcode/QR, warehouses, expiry,
batch tracking, minimum stock thresholds, supplier mapping, product
images, and fully configurable multiple units (kg, gram, piece, box,
packet, bundle, feet, meter, litre, dozen, carton, roll, and custom units).
The catalog itself does not need to exist up front — see Module 12.

### Module 3 — Billing
GST invoice, retail invoice, wholesale invoice, quotation, purchase bill,
credit note, debit note, delivery challan, estimate, recurring invoice,
partial/split payment, and UPI/cash/card/bank/credit payment methods.

### Module 4 — AI Billing
Create a bill from text ("10 Cement Bags, 5 PVC Panels, 2 Adhesives"),
voice, a photo, a PDF, a WhatsApp message, an email, OCR of a handwritten
note, or a scanned business card. Every input format normalizes into the
same structured invoice data.

### Module 5 — Price Intelligence
A master product catalog with prices that update from supplier PDFs,
Excel sheets, WhatsApp messages, websites, CSV, manual entry, or API. The
AI diffs old price vs. new price vs. margin vs. customer-specific pricing,
shows only what changed, and applies updates only after owner approval.

### Module 6 — Inventory Intelligence
Fast-moving products, dead stock, seasonal demand, stock aging, low
inventory alerts, overstock detection, and AI-suggested purchase
quantities based on predicted future demand.

### Module 7 — Supplier Management
Supplier profiles, purchase history, price trends, outstanding payments,
supplier rating, delivery performance, and automatic purchase order
generation.

### Module 8 — Expense Management
Rent, electricity, salary, transport, maintenance, internet, and
miscellaneous expenses, with monthly analytics and reporting.

### Module 9 — Business Dashboard
Today's sales, today's profit, monthly sales, top products, top customers,
pending payments, inventory value, expenses, cash flow, profit trends, and
a natural-language AI summary ("Sales are 12% higher than last week. PVC
Panels generated 48% of today's revenue. Adhesive stock runs out in 5
days.").

### Module 10 — AI Business Assistant
Conversational access to everything above instead of menu navigation:
"Show today's sales", "Which products are slow?", "Who hasn't paid?",
"Generate GST report", "Create purchase order", "Who bought PVC Panels
last month?"

### Module 11 — Connector Framework (Interoperability)
Nexora is designed to sit alongside the tools a business already uses,
starting with **Tally** (via Tally's XML/HTTP interface for two-way
voucher, ledger, and stock-item sync) and extending to Busy, Marg ERP,
Zoho Books, QuickBooks, and generic Excel/CSV import-export. See
[docs/integrations.md](docs/integrations.md).

### Module 12 — Catalog Intelligence (Self-Growing Catalog)
Solves the problem every competing app ignores: a real business never
starts with a clean, complete product list. The catalog is built
incrementally from every interaction — supplier bills, sales invoices,
photos, voice notes, handwritten lists, Excel/CSV — instead of a one-time
onboarding form. AI-detected products are staged, matched against the
existing catalog using fuzzy text, structured attributes, and semantic
similarity, and only become — or update — a real product after
confidence-appropriate confirmation, so the catalog never silently fills
with duplicates. Includes automatic SKU generation for businesses with no
SKU system, a `price_missing` state for products entered before their
price is known, and an AI Product Approval Queue for reviewing what the
AI detected. See [docs/product-catalog-lifecycle.md](docs/product-catalog-lifecycle.md).

## Artificial Intelligence Layer

The AI layer must understand: natural language, images, PDFs, voice,
Excel, CSV, emails, supplier catalogues, invoices, purchase bills,
delivery challans, and handwritten notes. See
[docs/ai-architecture.md](docs/ai-architecture.md) for the OCR, vision,
LLM, and speech pipeline design.

## Long-Term Ambition vs. Current Focus

Nexora's long-term destination is a full Business Operating System —
billing today, potentially accounting, payroll, CRM, and more over years.
That ambition is real and worth stating, but it is deliberately kept out
of this specification as anything other than a stated direction: the 12
modules above are what is actually being designed and built, scoped down
further to an even smaller V1 in [docs/mvp-scope.md](docs/mvp-scope.md).
Everything beyond these 12 modules lives in
[docs/future-modules.md](docs/future-modules.md), explicitly unscheduled,
so that long-term ambition never quietly expands what's being built right
now. See the Zoho comparison in that document for why this separation
matters.

## Long-Term Roadmap

See [docs/roadmap.md](docs/roadmap.md) for the full phase breakdown
(Phase 1 core operations through Phase 5 autonomous AI agent).

## Design Principles

- Mobile-first
- Fast (every workflow under 30 seconds)
- Minimal typing — AI should reduce input effort, not add to it
- Configurable, not hardcoded, per industry
- Trustworthy — the AI proposes, the owner approves financial changes
- Gradual adoption — modules can be turned on independently

## Success Metrics

- Invoice creation time under 30 seconds.
- Price updates completed in minutes instead of hours.
- Inventory accuracy above 99%.
- Less than 1% manual correction after AI extraction.
- A non-technical shop owner can learn the core flow in one day.

## Final Vision Statement

The objective is not to build another billing application. The objective
is to build an AI Business Operating System that acts as a digital
employee for any business — understanding natural language, documents,
images, voice, and business data to automate daily operations, reduce
manual work, improve accuracy, and help business owners make better
decisions, while interoperating with the billing and accounting software
(Tally and similar tools) businesses already depend on.
