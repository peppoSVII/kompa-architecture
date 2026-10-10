# 07 — Cost model: the AI is 30–66× the infrastructure

## The headline finding

Modelling this product at different volumes produced one number that reframed
every subsequent decision:

> **The AI is 30–66× the infrastructure cost.**

At 100k scans/month: **~$4.6 of hosting** against **~$307 of model usage**.
At that ratio, "which host is cheaper" is not a real question. Choosing between
two hosts that differ by a few dollars is noise; the model bill is the signal.

## The model

**Assumptions** (all stated, because a cost model without assumptions is a
guess):

| Assumption | Value |
|---|---|
| Instance size | 1 vCPU / 512 MB |
| Compute per scan | ~2.8 vCPU-seconds |
| Requests per scan | ~2 (job submission + result) |
| Image payload | ~300 KB |
| Egress price (EU) | $0.12 / GB |

| Volume / month | Hosting | Egress | **Infrastructure** | **AI** | Total |
|---|---|---|---|---|---|
| 1,000 | $0 (free tier) | ~$0.04 | **~$0** | $3 | ~$3 |
| 10,000 | $0 (free tier) | ~$0.4 | **~$0** | $30 | ~$30 |
| **100,000** | $0.96 | ~$3.60 | **~$4.6** | **$307** | **~$312** |
| 1,000,000 | ~$61 | ~$36 | **~$97** | **$3,070** | **~$3,167** |

## Where the model cost comes from

| Item | Price | Note |
|---|---|---|
| Input tokens | $0.30 / 1M | the image plus the prompt |
| **Output tokens** | **$2.50 / 1M** | ⚠️ **includes reasoning tokens** |

That second row is the important one. Output is **8× the price of input**, and
for a reasoning model the output count is not just "the JSON you asked for" — it
includes the tokens the model spends thinking. A prompt change that makes the
model reason longer is expensive in a way that is invisible if you only look at
the response size.

This is why **per-call token and cost logging** was built early: it is the only
way to attribute the bill to a feature, a prompt version, or a user. Without it,
cost optimisation is guesswork.

## Levers, ranked by expected effect

Applied in this order, because this is the order of magnitude of the effect:

| Lever | Direction |
|---|---|
| Reduce reasoning/output tokens | **largest** — output is 8× input and includes thinking |
| Right-size the model per task | large — not every extraction needs the same model |
| Reduce input: image size, prompt size, avoid re-sending context | medium |
| Batch / cache repeated work | medium |
| Choose a cheaper region or host | **small** — the ratio above says so |

## The guardrails that matter

A cost model is useless without a ceiling. The ones that apply here:

| Guardrail | Effect |
|---|---|
| **Spend cap on the model provider** | Blocks the line item that actually matters |
| **Budget alerts** at several thresholds | Early warning, before the cap |
| **Spend cap on the hosting platform** | Native: the platform pauses instead of billing |
| **Max instances** | A physical ceiling: requests are refused rather than scaled |
| **Request timeout** | No instance held hostage |
| **Per-user daily quota** | Persisted, atomic, so it survives restarts |
| **Per-user rate limit** | ⚠️ in-memory today — see [04](04-architecture-phase-2.md) |
| **Monthly API call budget on the places provider** | Keeps a secondary integration inside its free tier |

## The decision this drove

The hosting evaluation started as a cost question and stopped being one. Three
conclusions came out of it:

1. **Do not optimise the hosting bill.** At these volumes the difference between
   reasonable options is a few dollars. Optimise the model bill.
2. **The cheapest host is not automatically right.** The option that would have
   saved a couple of euros per month required an always-on process — which
   conflicted with the architecture in [04](04-architecture-phase-2.md). A cheaper bill that
   reintroduces in-memory state is a bad trade.
3. **Free tiers have conditions, not just limits.** One provider's free tier
   allowed using submitted content to improve their products. For a personal
   project that is tolerable; before a public release it is a privacy decision,
   not a cost decision. Worth reading the terms, not just the price.

## Sources

Figures are either **measured on this project** or **quoted from vendor pricing
pages** — and the two are labelled differently throughout. Unit prices for
compute are the ones I could not verify directly (the pricing page was too large
to fetch reliably); they were cross-validated against a documented real-world
bill and reproduced it exactly. That is a reasonable check, not a proof, and it
is stated as such.

→ Next: [08 — Decisions](08-decisions.md)
