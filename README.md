# Kompa — engineering case study

**What this repo is.** The architecture and decision record behind
[Kompa.app](https://www.kompa.app) — a production GenAI travel-expense app that I
designed, built and run end to end.

**What this repo is *not*.** It contains **no product source code**. Everything
here is reasoning: measurements, diagnoses, trade-offs, and the alternatives I
rejected. The product stays private; the engineering story does not.

**Why it exists.** Most portfolio repos show *what* was built. This one shows
*how problems were found* — including two I got wrong first, and the
measurements that proved it.

---

## The 60-second version

AI receipt scanning took **22–29 seconds**. The model was not the problem: the
host was **blackholing IPv6** to Google. Fixing it in code exposed a second
problem — the push system required an **always-on process**, which blocked
migrating to a host that scales to zero. So I moved the notification trigger
**into the client**, onto a durable outbox that already existed, and made
deduplication **atomic on the database**. The server became **stateless**.

Then I modelled the cost. At 100k scans/month the infrastructure is **~$4.6**.
The AI is **~$307**. The AI is **30–66× the infrastructure** — so that is where
the real engineering effort belongs, not in the hosting choice.

---

## What it demonstrates

| Skill | Where to look |
|---|---|
| Root-cause diagnosis from measurements, not guesses | [02 — The diagnosis](docs/02-diagnosis.md) |
| Designing for failure: durability, retries, idempotency | [04 — Going stateless](docs/04-stateless.md) |
| Threat modelling and API hardening | [05 — Security model](docs/05-security.md) |
| Unit economics and cost engineering | [06 — Cost model](docs/06-cost.md) |
| Documented trade-offs instead of opinions | [07 — Decisions](docs/07-decisions.md) |
| Testing without a framework: 7 suites, 265 assertions | [08 — Testing](docs/08-testing.md) |

## Index

| # | Document | In one line |
|---|---|---|
| 01 | [The problem](docs/01-problem.md) | Scans took 22–29 s. Here is what it was *not*. |
| 02 | [The diagnosis](docs/02-diagnosis.md) | An IPv6 blackhole between host and provider — and how it was isolated. |
| 03 | [Notification flow](docs/03-notifications.md) | From "a friend adds an expense" to "your phone rings" — and why it cannot be peer-to-peer. |
| 04 | [Going stateless](docs/04-stateless.md) | Removing the database listener, and the four bugs that came out of it. |
| 05 | [Security model](docs/05-security.md) | Threat model, the defences on the write path, and the gaps I have not closed yet. |
| 06 | [Cost model](docs/06-cost.md) | Unit economics with sources: where $1 actually goes. |
| 07 | [Decisions](docs/07-decisions.md) | Eight rejected alternatives, each with the reason. |
| 08 | [Testing](docs/08-testing.md) | Seven check suites, 265 assertions, no framework — and why. |

Diagrams live in [`diagrams/`](diagrams/).

---

## What this case study does *not* claim

Engineering write-ups are only useful if they are honest about the edges. So:

- **Not a success story with a tidy ending.** Two of the fixes below were
  wrong on the first attempt. Both are documented as such.
- **Not everything is finished.** The durable job queue and the migration to a
  scale-to-zero host are designed and planned, not shipped. Sections say so.
- **Known open gaps** are listed rather than hidden — including one that means
  push notifications currently work on one platform only
  ([05](docs/05-security.md), [07](docs/07-decisions.md)).
- **No numbers are invented.** Every figure is either measured on this project
  or quoted from vendor pricing pages, and the two are labelled differently.

## Context

Solo-built. Native iOS/Android client (React Native/Expo), a Node.js backend, and
Firebase Realtime Database as the single source of truth, with the Gemini API for
multimodal extraction. The project is also the subject of a CV entry, so this
document deliberately stops short of anything that is product rather than
engineering: no prompts, no pricing logic, no business rules.

---

## Licence

Proprietary, source-available for **reading only**. See [LICENSE](LICENSE).
You are welcome to reference this work and to ask me about it — not to reuse it.
