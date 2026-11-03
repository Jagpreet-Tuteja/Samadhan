# UI/UX Guidelines

## Why "ChatGPT meets Notion meets Stripe" is the wrong reference

That phrase describes a design language aimed at tech-forward, English-
fluent, high-bandwidth users — engineers, SaaS buyers, people comfortable
with whitespace-heavy minimalism and subtle affordances. It's a genuinely
good aesthetic for the wrong audience here. The actual buyer described
throughout [PROJECT_SPEC.md](../PROJECT_SPEC.md) — a hardware store,
medical store, or grocery store owner — is closer to a PhonePe, Google
Pay, or WhatsApp user: comfortable with simple, high-contrast, icon-led
interfaces, often on a mid-range Android phone, often on patchy data,
often not fluent in reading dense English UI copy, and not interested in
software aesthetics at all — only in whether it's fast and doesn't get in
the way. Designing for the Stripe/Notion crowd optimizes for the wrong
approval.

## Actual design references

PhonePe, Google Pay, WhatsApp Business, and (for the specific problem of
"make a complex backend feel simple") the better parts of Paytm for
Business — not Notion, not Linear, not Stripe Dashboard.

## Core principles

- **Large touch targets** (minimum ~48dp) — shop counters are used
  one-handed, often quickly, sometimes with a finger that isn't perfectly
  precise.
- **High contrast, works in bright light** — many of these devices get
  used near a shop's open storefront or under harsh fluorescent light,
  not a dim office. Avoid low-contrast grey-on-white patterns that read
  fine on a designer's monitor and disappear in direct sunlight.
- **One obvious primary action per screen.** Never make the owner
  scan a toolbar to figure out what to do next.
- **Minimal typography scale, minimal reliance on written English
  copy** — icons and structure carry meaning wherever possible; see
  regional-language input in [ai-architecture.md](ai-architecture.md) and
  [roadmap.md](roadmap.md) for the same principle applied to AI input,
  not just UI chrome.
- **No dense multi-column dashboards on mobile.** Progressive disclosure
  — show the three numbers that matter right now (today's sales, pending
  payments, low stock), let the owner drill in for more, rather than
  Module 9's full dashboard rendered at phone width by default.
- **Every state is explained, never a bare spinner.** Loading, saved,
  syncing, offline-queued (once [offline-strategy.md](offline-strategy.md)
  ships) — each gets a short, plain-language label, not just a spinner
  the owner has to trust blindly.
- **Speed is a design requirement, not a nice-to-have** — per
  [PRODUCT_PRINCIPLES.md](../PRODUCT_PRINCIPLES.md)'s 30-second workflow
  bar, every added tap, every extra confirmation screen, is a real cost
  against that bar.

## Where a more "modern SaaS" feel is actually appropriate

The Module 10 AI Assistant chat surface is the one place a more
ChatGPT-like conversational interface genuinely fits the interaction —
it's explicitly a conversation, and the audience opening it has already
chosen to type or speak a question rather than tap through a form. Keep
the chat-first aesthetic there; don't let it define the billing,
inventory, or dashboard screens that carry the actual daily workflow.

## Non-goal

Visual polish for its own sake, or design decisions made to impress a
technical/investor audience rather than the actual owner opening the app
at a shop counter. If a design choice would look great in a pitch deck
screenshot but adds friction to the 30-second invoice flow, the invoice
flow wins.
