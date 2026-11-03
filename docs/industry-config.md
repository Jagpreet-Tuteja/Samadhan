# Industry Configuration

Nexora supports many industries through configuration, not hardcoded
per-industry logic (see [PRODUCT_PRINCIPLES.md](../PRODUCT_PRINCIPLES.md)).

## What's configurable per organization

- **Units** — which units exist and their conversion ratios (e.g.
  hardware: piece/box/bundle; grocery: kg/gram/packet; textiles:
  metre/roll).
- **Tax rules** — applicable GST slabs per category, HSN/SAC codes,
  inclusive vs. exclusive pricing.
- **Document types in use** — a restaurant may never use purchase orders
  or delivery challans; a wholesaler relies on them heavily. Modules
  (Module 1–11) can be enabled/disabled independently per organization.
- **Price lists** — number and structure of price tiers (retail,
  wholesale, VIP, customer-specific).
- **Terminology** — e.g. "table" vs. "customer" for a restaurant,
  "patient" vs. "customer" for a medical store — surfaced as label
  overrides in the UI without changing the underlying data model.

## Onboarding flow (planned)

During signup, the owner picks an industry template (hardware, medical,
grocery, restaurant, wholesaler, "other"), which pre-fills sensible
defaults for units, categories, and tax rules — all editable afterward.
Templates are a starting point, not a constraint.

## Why this matters for the AI layer

Industry configuration is also grounding context for
[ai-architecture.md](ai-architecture.md): the LLM structuring step is
given the organization's actual units, categories, and terminology so
"5 kg" and "5 packets" resolve correctly for a grocery store, and "5
pieces" resolves correctly for a hardware store, without industry-specific
code paths.
