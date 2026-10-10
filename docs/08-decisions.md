# 08 — Decisions, alternatives, and what would change my mind

A decision without the rejected options is an opinion. Each entry records what
was chosen, what was rejected, **why** — and the condition under which the
choice should be revisited.

---

## D1 — Push notifications: client-driven, not server-observed

| | |
|---|---|
| **Chosen** | The client that performed the write reports it to the server |
| **Rejected** | A server process watching the database; database triggers invoking functions |
| **Why** | A watching process must always run, which blocks scaling to zero and keeps state in memory ([04](04-architecture-phase-2.md)). The client already knows everything the server would have to infer: user action vs automatic fill, add vs edit, mid-import vs finished. |
| **Cost of the choice** | Real work moved into the client (coalescing, outbox integration, guards) — a genuine trade, not a free win |
| **Would revisit if** | A writer appears that is *not* this client (another platform, an admin tool, a bulk import). That is exactly the bug in [04](04-architecture-phase-2.md), and it is the standing risk of this design. |

## D2 — Hosting: a scale-to-zero container platform

| Option | Monthly | Verdict |
|---|---|---|
| **Chosen: managed containers, scale-to-zero** | **~$4.6 @100k scans** | ✅ cheapest at volume *and* zero maintenance |
| Small dedicated VPS | €5.48 | ❌ similar price, but 15–30 min/month of maintenance forever |
| Container PaaS with a minimum instance | $8.78 | ❌ pays for idle capacity to avoid a 1–2 s cold start |
| Free-tier ARM instances | €0 | ❌ see D4 |

**Why:** at this volume it is both the cheapest and the least work. The deciding
factor was not the price — it was **maintenance time**, the only genuinely
scarce resource on a solo project.

**Would revisit if:** traffic becomes steady and high enough that reserved
capacity beats per-request pricing, or if the cold start becomes user-visible in
a flow that is *not* already asynchronous.

## D3 — Jobs: a durable queue, not work after the response

| | |
|---|---|
| **Chosen** | Enqueue a job, respond immediately, process it as a tracked request |
| **Rejected** | Fire-and-forget work after sending the response (the current state) |
| **Why** | Work performed after a response is invisible: if the process is recycled mid-job, the work is silently lost and the user waits forever. A queue makes it tracked, retryable and observable — and it is what makes a scale-to-zero host safe for long jobs. |
| **Status** | ⚠️ **designed, not shipped** — the current implementation is still fire-and-forget |
| **Would revisit if** | Jobs become short enough (a few seconds) that the recycling risk is negligible. They are not. |

## D4 — Not free-tier ARM instances, despite €0

| Risk | Detail |
|---|---|
| Reclamation for inactivity | Instances below a CPU threshold over a week **can be reclaimed** |
| Capacity | Free ARM instances come from a shared, non-guaranteed pool |
| No SLA | For a production app holding personal data |
| Limits cut without notice | Allocation was reduced once, mid-year, with no prior announcement |
| Permanent home region | Cannot be changed later |

**Verdict:** fine for a hobby, not for something with real users. €0 is not a
price if the downside is "your app disappears and cannot be moved".

## D5 — Not a request-scoped edge runtime (for now)

| Objection | Status |
|---|---|
| A request-scoped runtime cannot hold long-lived listeners | ✅ was decisive — ❌ **no longer applies** |
| The server SDK does not run there: token verification, database access and push signing would all need reimplementing | ✅ still applies |
| The rest of the backend would need porting to a different runtime | ✅ still applies |

**Honest note:** the first objection was the real blocker, and the work in
[04](04-architecture-phase-2.md) **removed it**. The decision should be re-examined rather
than inherited — what remains is porting cost, not feasibility. A decision that
was correct under old constraints is not automatically correct under new ones.

## D6 — Not database-triggered functions, either

| Objection | Status |
|---|---|
| Requires a paid plan | ✅ still applies |
| Would require rewriting in-memory state onto shared storage | ❌ **no longer applies** — there is no in-memory state left |
| Per-invocation rather than per-request cost model | ✅ still applies |

Same honesty as D5: the strongest objection has expired. What remains is a
cost-and-control comparison, not a feasibility one.

## D7 — Data path: direct client access, protected by rules

| | |
|---|---|
| **Chosen** | The client reads and writes the database directly; authorisation is enforced by **versioned security rules** |
| **Rejected** | Routing every read and write through the backend |
| **Why** | It is the platform's intended model, it removes a hop from every interaction, and it makes realtime sync trivial. It is **not** an anti-pattern — the anti-pattern is a client writing with no rules. |
| **Supporting decision** | The rules live in the repository, versioned, not hand-edited in a console. Unversioned rules drift, and drift in a security boundary is a silent failure. |
| **Would revisit if** | Business logic needs enforcing on writes that the rules cannot express. So far it does not. |

## D8 — Effects belong on the server

The dividing line, applied consistently:

> **Data → direct from the client** (protected by rules).
> **Side effects → the server** (push, AI, quotas, privileges).

Every "should the client do this?" question was answered by asking whether it is
*data* or a *side effect*. This is the single rule that produced most of the
other decisions in this document.

---

## Why this section exists

None of these decisions is permanent, and two of them (D5, D6) are explicitly
flagged as **expired reasoning**. That is the point: recording the *condition*
attached to a decision is what makes it revisitable instead of tribal knowledge.

→ Next: [09 — Testing](09-testing.md)
