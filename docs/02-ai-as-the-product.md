# 02 — AI as the product, not as a feature

## Where AI belongs here

The reason a shared-expense app dies is rarely the arithmetic. It is the
**typing**: splitting one supermarket receipt by hand is five minutes of data
entry, and doing that after every meal is why the ledger is abandoned in week
one.

So AI appears at exactly one kind of place in Kompa — **where it removes typing**:

| Capture surface | What the model produces |
|---|---|
| A photo of a receipt | line items with prices, ready to be assigned |
| A booking, ticket or travel document | a single itinerary stop |
| A **screenshot of a plan** | **many stops at once**, previewed before saving |
| A photo or a **PDF** | **many expenses at once** |
| A sentence, typed or **spoken** | one expense, or one stop |

Every one of them ends in the same place: structured rows the user can correct
before anything is written. **The model proposes; the user confirms.** That is a
deliberate position — an extraction that silently writes to a shared ledger is a
much worse product than one that shows its work first.

## The rule: the key never ships

The model is never called from the client. Requests go through the backend,
which enforces, in order:

| Check | Why |
|---|---|
| **Firebase ID token** | the caller is a known user, and the user is taken from the token |
| **Per-user rate limit** | one account cannot hammer the endpoint |
| **Per-user daily quota** | a bad day cannot become a bill |
| **Request size cap** | images are bounded before they reach the model |
| **Allowed content types** | images and PDFs only |
| **Per-call logging** | every call is attributable — see below |

This was not the first design. The native client initially carried the model key
in its bundle, and removing it — reintroducing the proxy, then rotating the
exposed key — was the first item on the release checklist, ahead of every
feature. It is the correct order: **a mobile binary is downloadable, so a
shipped key is a published key.**

The same reasoning as [06](06-security.md), applied to a different secret.

## Why the model is a configuration detail

Because everything goes through one proxy, the provider is an implementation
choice rather than a product decision: it is selected by an environment
variable, and the client only ever sees `/api/messages`. Swapping models is a
deployment, not a release.

That is also what makes the cost work in [07](07-cost.md) possible: there is one
choke point through which every token passes, so there is one place to count.

## Cost discipline as a feature

Every AI call is logged with the user, the trip, the **feature** it served
(`OCR_RECEIPT`, `QUICK_ADD_EXPENSE`, `PLAN_CITY`), the model, the prompt and
completion tokens, and an estimated cost.

Without that, a cost model is a guess. With it, three questions become
answerable: *which feature spends the money*, *which prompt version changed the
bill*, and *what does one scan actually cost*. The answer to the last one —
measured, not estimated — is in [07](07-cost.md), and it reframed the whole
hosting discussion.

## AI is what makes the join real

The product thesis in [01](01-what-kompa-is.md) is that the plan and the money
are one object. AI is what makes that practical instead of aspirational:

- a **screenshot of a plan** becomes several itinerary stops;
- a **booking** becomes a stop with a date and a time;
- a **receipt** becomes expenses, each attachable to a stop (`stopId`);
- a **spoken sentence** becomes either.

One capture surface, feeding both halves. Without extraction, the join would
exist in the data model and nobody would ever populate it.

## What the model is *not* asked to do

- **It does not compute money.** Splitting, settlement and balances are pure
  functions in the client ([01](01-what-kompa-is.md)), identical on both
  clients, and covered by tests ([09](09-testing.md)). A language model is a
  reasonable extractor and an unacceptable accountant.
- **It does not hold business rules.** No prompts, no pricing logic and no
  business rules are published in this repository — deliberately, as stated in
  the [README](../README.md).

## Honest edges

| Edge | Consequence |
|---|---|
| Extraction accuracy depends on image quality | manual editing is always available, and the preview step exists for exactly this |
| The provider's free tier may use submitted content | acceptable for a personal project; a **privacy decision** before a public release — flagged in [07](07-cost.md) |
| The reasoning budget of a model is invisible in the response size | output tokens include thinking, and output is the expensive side — see [07](07-cost.md) |

→ Next: [03 — Architecture, phase 1](03-architecture-phase-1.md)
