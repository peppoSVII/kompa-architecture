# 01 — What Kompa is

## The gap it targets

Travel is a fragmented market. One trip is normally spread across four unrelated
tools — one for the money, one for the plan, one for the flights, one for
inspiration — each with its own account, its own notifications, and its own idea
of what "the trip" is.

Kompa joins two of those halves, which are almost always separate:

> **What a group plans to do** and **what it spends doing it** are one object,
> not two.

That sentence is the product thesis. It is also why the architecture in
[03](03-architecture-phase-1.md) and [04](04-architecture-phase-2.md) looks the
way it does: the join is not a screen, it is a **data relationship**.

## Where it started: one file

The first version was a **single self-contained `index.html`** — React 18 and
Tailwind loaded from a CDN, no build step, no bundler, no backend. Open it in a
browser and it works; send it as a file and it still works.

It solved one problem well: photograph a receipt, let the model extract the
items, then assign each item to one of two people by swiping left, right or to
the centre, and read the settlement. The constraint was deliberate — one file
means no install, no deployment and no infrastructure to maintain, which is
exactly the friction budget a side project has.

That file is still in the product repository, and it is still the **reference
implementation of the behaviour**: the native client is expected to match it.

## What it became: the same product, natively

The PWA was a validation step, not the destination. The product now also ships
as a **native iOS/Android client** (Expo SDK 57, `expo-router`, React Native
0.86, TypeScript, New Architecture) — and the two clients are not two products.

The decision that keeps them one product was to **port the pure logic verbatim**
rather than reimplement it:

| Module | The decisions it owns |
|---|---|
| `money.ts` | splitting, settlement, group debts, trip metrics, stable person keys, rounding |
| `i18n.ts` | every user-facing string — one key set, three languages |
| `currency.ts` | currencies, expense and itinerary categories, conversion |

Both clients therefore compute the same numbers and speak the same three
languages. Nothing in that table touches the network, the UI or the platform —
which is what makes it portable, and testable ([09](09-testing.md)).

## The two halves, joined

The join is one field, plus the modules built around it:

| Piece | What it does |
|---|---|
| `stopId` on an expense | attaches a cost to a specific itinerary stop |
| `stopExpense` | the rules that decide what a stop has cost so far |
| `stopReminders` / `reminderPlan` | reminders derived from the stops that carry a time |

So a stop can show what was spent on it, a trip can carry a budget, and the
budget can be read per day. The itinerary is not decoration next to the ledger:
it is the axis the ledger is organised by.

## What it does

- **Shared trips** — a group, an invite link, one ledger.
- **Expenses** — split line by line, or divided evenly; settlements and balances.
- **Multi-currency** — foreign spend converted, balances always readable.
- **Itinerary** — day-by-day stops with dates and times, on a map, with place search.
- **AI capture** — a receipt, a booking, a screenshot of a plan, a PDF, a spoken
  sentence ([02](02-ai-as-the-product.md)).
- **Shareable recap** — a read-only snapshot of the trip, inside the app.
- **Offline** — writes are journaled locally and replayed; the app is usable with no signal.
- **Three languages** — Italian, English, Spanish.

## Who it is for

The beachhead is deliberately narrow: **a group of friends coordinating one
trip** — the situation where the shared ledger and the shared plan are the same
conversation. A solo traveller is a later segment, not the design centre.

## Where it sits

Being honest about the incumbents is more useful than pretending they are absent:

| Player | Their real advantage | Where they are exposed |
|---|---|---|
| **Splitwise** | the network effect — your friends already have it installed | moving features behind a subscription |
| **Wanderlog / Lambus** | travel SEO and visual planning | offline and email import behind a paywall; premium does not extend to invited guests |
| **Google** (Maps Timeline + Travel + Gemini) | distribution — and it can copy most planning features for free | friction, and integrated *group* accounting, which it does not play |

The conclusion that shaped the product is uncomfortable but useful:

> **The AI is not the differentiator.** In 2026, every product has one.

What is genuinely hard to copy is the **group loop**: the organiser creates the
trip and shares a link, and the other people join with minimal friction. That
loop is both the growth channel and the most defensible asset — and it is a
*systems* property, not a feature: it constrains identity, permissions, realtime
sync and offline behaviour at once.

## What that means for the engineering

A group-scoped, realtime, offline-tolerant product with a tight friction budget
produces the constraints the rest of this case study is about:

1. **AI where it removes friction, not where it looks impressive** → [02](02-ai-as-the-product.md)
2. **A backend that cannot be removed, and must stay cheap** → [03](03-architecture-phase-1.md)
3. **State that survives scale-to-zero, and pushes that arrive exactly once** →
   [04](04-architecture-phase-2.md)
4. **The bugs that fell out of all of it** → [05](05-problems-and-fixes.md)

→ Next: [02 — AI as the product](02-ai-as-the-product.md)
