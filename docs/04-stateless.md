# 04 — Going stateless, and the four bugs it exposed

## Why the server had to change

The push system originally worked by **listening to the database**: a process
kept subscriptions open on the collections that matter, and pushed when
something changed.

That design has one non-negotiable requirement: **the process must always be
running.** Which means:

- it cannot run on a host that scales to zero;
- every restart loses whatever it was holding in memory;
- on a free tier that suspends idle services, it silently stops working.

The migration goal was a host that scales to zero. That made the listener the
blocking dependency — not the AI, not the database, not the client.

## The in-memory state that had to go

Six pieces of state lived in that process:

| State | What it was for |
|---|---|
| Boot timestamp + grace window | Ignore pre-existing data on startup |
| A "last seen" map | Detect what actually changed |
| A "linked" map | Distinguish "person added" from "person joined" |
| A pending batch + window | Coalesce bursts into one push |
| A dedup cache | Avoid duplicate pushes |
| A timer per group | Schedule the flush |

All of it was correct — and all of it was **fatal for horizontal scaling**:
with two instances, each holds its own half-truth, and the dedup cache stops
deduplicating.

## What replaced it

| Before | After |
|---|---|
| A server watching the database | The **client that performed the write** reports it |
| In-memory dedup cache | **Atomic transaction on the database** (shared, instance-safe) |
| In-memory batch window | Coalescing **in the client**, before it sends |
| A timer inside a live process | A **durable on-disk outbox** with retries |

The trigger moved to the only participant that already knows everything: the app
that made the change. It knows whether this was a user action or an automatic
fill, an add or an edit, and whether it is mid-way through importing ten photos.

And the intent rides the **write outbox that already existed**, so durability,
retry and replay were already solved.

## Bug 1 — duplicate pushes

**Symptom:** during the transition, both the old listener and the new
client-driven path were active. The same event produced two pushes.

**Fix:** deduplication was moved from an in-memory cache to an **atomic
transaction on the database**, keyed by group and event type, with a five-second
window. Two instances racing the same event now resolve deterministically, and
the loser reports `skipped` instead of sending.

**The lesson:** "in-memory is fine" is a statement about *how many instances you
run*. It was true when written and false by the time it mattered.

## Bug 2 — notifications for records that no longer exist

**Symptom:** a notification was queued, the record was deleted before the queue
flushed, and the server correctly refused to notify about a missing record. The
client retried forever, because "missing" looked like "not synced yet".

**Fix:** deleting a record now also **purges its pending intent** from the queue
and from the outbox, so a doomed request is never sent.

**The lesson:** a retryable error needs a terminal condition, or it is an
infinite loop with extra steps.

## Bug 3 — writes that were never announced

**Symptom:** a legacy web client wrote to the same collections through a
different code path. Those writes never produced a notification, because the
notification logic lives in the current client.

**Fix:** the legacy client was disabled in production, and its service worker
turned into a kill-switch that uninstalls it from devices that still have it.

**The lesson:** the moment you move a responsibility into the client, *every
client* becomes part of the contract. "The app" is not one program.

## Bug 4 — old notifications appearing when the app opened

This one was the most interesting, because the first diagnosis was wrong.

**Symptom:** opening the app sometimes produced notifications about expenses
created *days* earlier by other members.

**First hypothesis:** stale entries in the outbox. Wrong — the outbox was
correctly drained.

**Actual cause 1 — a double emit.** The subscription helper emits **twice**: first
the value from the **on-disk cache**, then the value from the **network**. The
change detector treated the first emit as its baseline. When the cache was stale
or empty, the second emit looked like "everything is new" — and every record
missing from the cache produced a notification.

That also explains the intermittency: it depended on how stale the cache was.

**Actual cause 2 — an unrelated one.** Push messages are **queued** by the
delivery channel when a device is offline. With no expiry, a message could be
delivered days later, when the device reconnected. So two independent mechanisms
produced the same user-visible symptom.

**Fixes:**
1. Two guards were reintroduced in the client — the same two the old server
   listener had: a **warm-up window** after subscribing (covers the cache→network
   transition) and a **recency threshold** on timestamps (covers late diffs).
   Both were extracted into a small pure module so they could be unit-tested.
2. Push messages carry a **one-hour TTL**, so a stale notification is dropped
   rather than delivered late.

**The lesson:** when two mechanisms can produce the same symptom, fixing one
leaves the bug alive and the diagnosis wrong. The symptom was "old
notifications"; the causes were unrelated.

## What is still stateful

Honest inventory of what remains in memory on the server:

| State | Status |
|---|---|
| Per-user daily quota | ✅ persisted, atomic transaction on the database |
| **Per-user rate limit** | ⚠️ **still an in-memory map** |

The rate limit is the last piece of instance-local state. It is bounded (the
worst case is a limit multiplied by the number of instances) but it is
inconsistent with the rest of the design, and it is on the list before the
scale-to-zero migration ships.

→ Next: [05 — Security model](05-security.md)
