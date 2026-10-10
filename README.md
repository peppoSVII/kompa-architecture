# Kompa — engineering case study

**What this repo is.** The architecture and decision record behind
[Kompa.app](https://www.kompa.app) — a live travel companion that keeps a group's
**plan** and a group's **money** in the same place. I designed and built the
engineering side end to end: the web client, the native client, the backend, the
notification system, the security model and the cost analysis.

**What this repo is *not*.** It contains **no product source code**, and no
prompts, pricing logic or business rules. Everything here is reasoning:
measurements, diagnoses, trade-offs, and the alternatives I rejected.

**Why it exists.** Most portfolio repos show *what* was built. This one shows
*how the problems were found* — including two I got wrong first, and the
measurements that proved it.

---

## The 60-second version

**The product.** Travel is fragmented: one tool for the money, one for the plan,
one for the flights. Kompa joins two of those halves — *what a group plans to do*
and *what it spends doing it* — into a single object. It started as one
self-contained HTML file and became a native iOS/Android app sharing the same
pure logic. AI is used where it removes typing, never where it looks impressive.
[→ What Kompa is](docs/01-what-kompa-is.md)

**The architecture, in two phases.** Phase 1 was the honest small-team shape: one
Node process that proxied the model, ran extraction jobs, watched the database to
send group notifications, and served the site — cheap, quick to build, and
carrying three structural debts. Phase 2 removed the debt: the notification
trigger moved into the client, onto a durable outbox that already existed,
deduplication became an atomic transaction on the database, the server became
**stateless**, and the target host became a scale-to-zero container platform whose
idle bill is **€0**.
[→ Phase 1](docs/03-architecture-phase-1.md) · [→ Phase 2](docs/04-architecture-phase-2.md)

**The measurement that started it.** AI extraction took 22–29 seconds in
production. It was not the model, the prompt, the image or the quota: it was the
host's **IPv6 path**, silently blackholed. One line of code fixed it — and *where
that line lives* turned out to matter more than the fix itself.
[→ Problems and fixes](docs/05-problems-and-fixes.md)

---

## What it demonstrates

| Skill | Where to look |
|---|---|
| Product thinking: where AI belongs, and where it does not | [02 — AI as the product](docs/02-ai-as-the-product.md) |
| Root-cause diagnosis from measurements, not guesses | [05 — Problems and fixes](docs/05-problems-and-fixes.md) |
| Evolving an architecture without a rewrite — two phases, one product | [03](docs/03-architecture-phase-1.md) · [04](docs/04-architecture-phase-2.md) |
| Designing for failure: durability, retries, idempotency, at-most-once delivery | [04](docs/04-architecture-phase-2.md) |
| Threat modelling and API hardening | [06 — Security model](docs/06-security.md) |
| Unit economics and cost engineering | [07 — Cost model](docs/07-cost.md) |
| Documented trade-offs instead of opinions | [08 — Decisions](docs/08-decisions.md) |
| Testing without a framework: 7 suites, 265 assertions | [09 — Testing](docs/09-testing.md) |

## Index

| # | Document | In one line |
|---|---|---|
| 01 | [What Kompa is](docs/01-what-kompa-is.md) | The fragmented market, the two halves joined, and why the AI is not the moat. |
| 02 | [AI as the product](docs/02-ai-as-the-product.md) | Where extraction belongs, why the key never ships, and how AI makes the join real. |
| 03 | [Architecture, phase 1](docs/03-architecture-phase-1.md) | One process on Render: what it bought, and the three debts it left behind. |
| 04 | [Architecture, phase 2](docs/04-architecture-phase-2.md) | Stateless, client-driven push, and a host that scales to zero. |
| 05 | [Problems and fixes](docs/05-problems-and-fixes.md) | Twelve problems, each as symptom → cause → fix → lesson. |
| 06 | [Security model](docs/06-security.md) | Threat model, the defences on the write path, and the gaps I have not closed yet. |
| 07 | [Cost model](docs/07-cost.md) | Unit economics with sources: where $1 actually goes. |
| 08 | [Decisions](docs/08-decisions.md) | Eight rejected alternatives, each with the reason. |
| 09 | [Testing](docs/09-testing.md) | Seven check suites, 265 assertions, no framework — and why. |

Diagrams (Mermaid — rendered by GitHub):
[system overview, phase 1 → phase 2](diagrams/system-overview.md) ·
[the travel ↔ expense bridge](diagrams/travel-expense-bridge.md)

---

## What this case study does *not* claim

Engineering write-ups are only useful if they are honest about the edges:

- **Not a success story with a tidy ending.** Two of the fixes in
  [05](docs/05-problems-and-fixes.md) were wrong on the first attempt. Both are
  documented as such.
- **Not everything is finished.** The durable job queue and the migration to a
  scale-to-zero host are designed and planned, not shipped. The sections say so.
- **Known open gaps are listed rather than hidden** — a per-user rate limit still
  held in memory, and push notifications that currently work on one platform only
  ([04](docs/04-architecture-phase-2.md), [06](docs/06-security.md)).
- **A deliberate trade-off is visible in the product.** The web client was the
  friction-reduction bet; it was disabled in production so that client-driven
  notifications could be trusted, and its service worker became a kill-switch.
  The shareable recap therefore renders inside the app, not in a browser
  ([04](docs/04-architecture-phase-2.md)).
- **No numbers are invented.** Every figure is either measured on this project or
  quoted from vendor pricing pages, and the two are labelled differently.

## Context

Built by a **two-person founding team**. Everything in this repository is my own
engineering work: the web and native clients, the backend, the notification
system, the security model and the cost analysis.

Stack: a single-file React web client and a React Native / Expo native client, a
Node.js backend, Firebase Realtime Database as the single source of truth, and a
multimodal model for extraction behind a server-side proxy.

This repository is deliberately separate from the product repository: the product
is private, this one is public and contains only the engineering story.

---

## Licence

Proprietary, source-available for **reading only**. See [LICENSE](LICENSE).
You are welcome to reference this work and to ask me about it — not to reuse it.
