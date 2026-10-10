# 02 — The diagnosis: an IPv6 blackhole

## The mechanism

The host resolved the AI provider's hostname to **IPv6 first**, tried to connect
over IPv6, and the packets were **silently dropped**.

"Silently" is the whole problem. A refused connection returns an error
immediately. A *blackholed* connection returns nothing at all — so the client
sits there until its own timeout expires. No exception, no error log, no trace
of it in the application.

Then the request was retried, and **each retry paid the same timeout again**.
That is how a network fault becomes a 29-second user-visible delay.

```
resolve hostname
   → returns IPv6 address first        ← Node's default order
   → TCP connect over IPv6
        → packets dropped, no RST      ← the blackhole
        → wait for connect timeout     ← ~10 s, nothing logged
   → retry                             ← pays it again
   → ...                               ← total: 22–29 s
```

The application code was never involved. It was blocked in a `connect()` call.

## Why the default behaviour bit us

Modern DNS resolvers return records in the order the upstream provides, and
IPv6 is commonly first. Some runtimes implement **Happy Eyeballs** — racing
IPv4 and IPv6 in parallel and taking whichever answers first. Others simply
trust the order they are given.

When the given order points at a path that drops packets, "trust the order" is
the worst possible strategy: it is not a race, it is a single attempt at the
address most likely to fail.

## The fix — and why it went in the code

```js
// server.js — first line of network setup
require('node:dns').setDefaultResultOrder('ipv4first');
```

One line. It makes name resolution return **IPv4 addresses first**, so the
connection goes down the path that works.

**The important part is *where* the fix lives.** The obvious alternative is to
change the host's network configuration. I deliberately did not, for three
reasons:

| Reason | Detail |
|---|---|
| **It travels with the code** | The fix applies wherever the service runs. A configuration fix applies only to the machine it was applied to. |
| **It is reviewable** | It is in a diff, with a comment explaining why. Config changes leave no such trace. |
| **It cannot silently regress** | The next migration would have re-introduced the bug. A future maintainer would have no reason to look for it. |

That last point is not hypothetical: the project's next step is migrating to a
different host. A config-only fix would have hidden a landmine inside that
migration.

## Result

| | Before | After |
|---|---|---|
| Scan latency (production) | 22–29 s | **~3.5 s** |
| Path to a working connection | IPv6 (blackholed) | IPv4 |

The scan is now bound by the model, which is where the time should have been
spent all along.

## The transferable lesson

When a distributed call is slow, and the application is not doing anything
expensive, **stop looking at the application**. Compare the same call from two
different origins. If the answer changes, the problem is the path — and the path
is usually fixable in code, durably, in one line.

→ Next: [03 — Notification flow](03-notifications.md)
