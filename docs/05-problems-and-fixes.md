# 05 — Problems, and how they were solved

## Why this document is the case study

Features are the easiest part to describe and the easiest part to copy. What is
hard to copy — and what a portfolio is actually for — is **how the problems were
found**. Two of the entries below were diagnosed wrong the first time, and they
are written up that way.

Every entry follows the same shape: symptom → cause → fix → what it taught.

## All of them, at a glance

| # | Problem | Status |
|---|---|---|
| 1 | AI extraction took **22–29 s** in production | ✅ fixed in one line of code |
| 2 | **Dead push tokens** accumulated forever | ✅ pruned at three points in the send path |
| 3 | **Security rules were not versioned** | ✅ rules and config live in the repository |
| 4 | **Jobs were fire-and-forget** | ⏳ open — Cloud Tasks ([04](04-architecture-phase-2.md)) |
| 5 | The notification listener kept **six pieces of in-memory state** | ✅ listener removed (594 → 410 lines) |
| 6 | **AI cost concentrated in the output** (82%) | ⏳ open — levers ranked in [07](07-cost.md) |
| 7 | On the free tier, listeners **died after 15 minutes idle** | ✅ superseded — the listener no longer exists |
| 8 | The listener's filtering rules had to be **reimplemented in the client** | ⚠️ standing risk — mitigated with pure modules + tests |
| 9 | **Old notifications** appeared when the app opened | ✅ fixed — see below |
| 10 | **Duplicate pushes** during the migration | ✅ atomic dedup on the database |
| 11 | Notifications for records that **no longer existed** retried forever | ✅ deleting a record purges its pending intent |
| 12 | Writes from the **legacy web client** were never announced | ✅ PWA disabled in production + kill-switch in its service worker |

---

## 1 · Extraction that took 22–29 seconds

### Symptom

A user photographs a receipt; the app sends it to the backend; the backend calls
the model; structured JSON comes back. It took **22–29 seconds** — for a flow
whose entire promise is "point your camera and forget about it". Users assumed
the app had hung.

### What it was *not*

Ruled out in this order, each one by **measurement rather than reasoning**:

| Hypothesis | How it was tested | Result |
|---|---|---|
| The model is slow | same image and prompt, called directly | ✗ |
| The prompt is too large | token count logged per call | ✗ |
| The image is too big | resized from ~3 MB to ~300 KB | ✗ |
| My code is slow | every stage timed server-side | ✗ |
| The API quota is throttling | usage counters checked | ✗ |
| **The host's network path** | **the same request from two origins** | ✅ **this one** |

### The measurement that broke it open

| Origin (same key, same payload, same model) | Time to reach the provider |
|---|---|
| The production host | **29 246 ms** |
| The same request from a laptop | **3 500 ms** |

An **8× difference that depended only on where the request came from**. That
eliminated the application, the model and the prompt in a single comparison and
left one variable: the path. A supporting signal made it sharper — the service
answered inbound requests in **657 ms**, so ingress was fast and egress was broken.

### Cause: an IPv6 blackhole

The host resolved the provider's hostname to **IPv6 first**, tried to connect,
and the packets were **silently dropped** — an **IPv6 blackhole**: no reset, no
error, nothing logged. The client waited for its own connect timeout, then
retried, paying the timeout again. The application code was never involved; it
was blocked inside `connect()`.

### Fix

One line, at the top of the server:

```js
require('node:dns').setDefaultResultOrder('ipv4first');
```

**Where the fix lives matters as much as the fix.** The obvious alternative was
to change the host's network configuration. That was rejected: a config fix
applies only to the machine it was applied to, leaves no trace in a diff, and —
decisively — the next host migration would have re-introduced the bug with
nothing to warn anyone.

### Result

| | Before | After |
|---|---|---|
| Scan latency (production) | 22–29 s | **1.48–3.01 s** (6 consecutive samples, average 2.27 s) |

### What it taught

The expensive mistake was never the slow scan; it was the risk of **optimising
the wrong layer** — rewriting prompts and shrinking images, real work that would
have addressed nothing. When a distributed call is slow and the application is
not doing anything expensive, stop reading the application: **issue the same call
from two origins**. If the answer changes, the problem is the path.

## 2 · Notifications about expenses from days ago

### Symptom

Opening the app sometimes produced notifications about expenses created **days
earlier** by other members.

### First hypothesis, and why it was wrong

Stale entries in the outbox. Wrong — the outbox was draining correctly.

### Cause 1 — a double emit

The subscription helper emits **twice**: first the value from the **on-disk
cache**, then the value from the **network**. The change detector treated the
first emit as its baseline, so when the cache was stale or empty the second emit
looked like "everything is new" — and every record missing from the cache
produced a notification. That also explains the intermittency: it depended on
how stale the cache happened to be.

### Cause 2 — an unrelated one, producing the same symptom

Push messages are **queued** by the delivery channel when a device is offline.
With no expiry, a message could be delivered days later, when the device
reconnected.

### Fixes

1. Two guards were reintroduced in the client — the same two the old server
   listener had: a **warm-up window** after subscribing (covers the
   cache→network transition) and a **recency threshold** on timestamps (covers
   late diffs). Both were extracted into a small pure module so they could be
   unit-tested ([09](09-testing.md)).
2. Push messages carry a **one-hour TTL**, so a stale notification is dropped
   instead of delivered late.

### What it taught

When **two mechanisms can produce the same symptom**, fixing one leaves the bug
alive and the diagnosis wrong. The symptom was "old notifications"; the causes
were unrelated.

---

## 3 · Delivery guarantees, and the three ways they broke

| Symptom | Cause | Fix |
|---|---|---|
| The same event produced **two pushes** during the migration | deduplication lived in an in-memory cache, and two instances hold two half-truths | **atomic transaction on the database**, keyed by group and event type, 5 s window — the loser reports `skipped` |
| A notification for a **deleted record** retried forever | "missing" is indistinguishable from "not synced yet", so the client kept retrying a doomed request | deleting a record now **purges its pending intent** from the queue and the outbox |
| Writes from the **legacy web client** were never announced | the notification logic had moved into the current client, and the old one was still installed on devices | the PWA was **disabled in production**, and its service worker became a **kill-switch** that uninstalls it |

Three lessons, in increasing order of usefulness:

1. **"In-memory is fine" is a statement about how many instances you run.** It
   was true when written, and false by the time it mattered.
2. **A retryable error needs a terminal condition**, or it is an infinite loop
   with extra steps.
3. **The moment a responsibility moves into a client, every client becomes part
   of the contract.** "The app" is not one program.

---

## The lessons that generalise

| Lesson | Where it came from |
|---|---|
| Measure before optimising: compare the same call from two origins | 1 |
| Fix the path in **code**, so the fix travels with the deployment | 1 |
| Two mechanisms, one symptom — fix both or misdiagnose both | 2 |
| A retryable failure needs a terminal condition | 3 |
| In-memory state is a statement about instance count | 5, 10 |
| Every client version in the wild is part of the contract | 12 |
| A free tier is a set of conditions, not just a set of limits | 7 |

→ Next: [06 — Security model](06-security.md)
