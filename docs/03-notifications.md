# 03 — Notification flow: from a write to a ringing phone

## The question that shapes the design

*Can device A make device B ring, without a server in the middle?*

**No — and it is worth being precise about why**, because the reason determines
the whole architecture.

## The fundamental constraint

| Platform | What happens when the app goes to the background |
|---|---|
| iOS | The system **suspends** it within seconds: no CPU, no open socket, no running code |
| Android | Same, via background execution limits and doze |

A suspended process cannot be contacted. There is nothing to connect to, no port
to knock on, no code to receive a packet.

So peer-to-peer delivery is impossible **by platform design**, not by choice.
And a paradox closes the question: a direct device-to-device link would only work
if *both* devices were online at the same moment — which is exactly when the
notification is unnecessary, because the user is already looking at the app.

## The only door that remains

| Platform | Channel | Who runs it |
|---|---|---|
| iOS | APNs | Apple |
| Android | FCM | Google |

These channels hold a connection that belongs to the **operating system**, not
to the app — which is precisely why they can wake an app that is not running.

And that channel demands a credential that **cannot ship inside the app**: a
service account, or an APNs key. Whoever holds it can send notifications to
**every user of the app**, in the app's name. It is not per-user revocable. A
mobile binary is downloadable and decompilable, so a shipped credential is a
published credential.

**Therefore exactly one hop through a server is structurally required.** That
server is the only place the credential can live.

## The flow

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. DEVICE A — the writer                                             │
│    A user adds an expense                                            │
└───────────────────────────────┬──────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. APP — write path                                                  │
│    • writes the expense to the database  ───────►  source of truth   │
│    • appends an intent to the ON-DISK outbox: "notify group G"       │
└───────────────────────────────┬──────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. APP — coalescing window (~3.5 s)                                  │
│    Importing 10 photos produces ONE notification, not ten            │
└───────────────────────────────┬──────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. APP — durable outbox                                              │
│    Survives app termination. Retried on failure and on reconnect.    │
└───────────────────────────────┬──────────────────────────────────────┘
                                │  POST /api/notify  +  ID token
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 5. SERVER — the trust boundary                                       │
│    • who are you?        → from the TOKEN, never the payload         │
│    • are you in group G? → checked against the database → 403 if no  │
│    • does it exist?      → re-read from the database    → 409 if no  │
│    • already sent?       → atomic dedup, 5 s window     → skip       │
└───────────────────────────────┬──────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 6. DATABASE                                                          │
│    Members of group G, their device tokens, their language           │
└───────────────────────────────┬──────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 7. FCM / APNs                                                        │
│    Delivery. If a device is offline the message is queued — until    │
│    it expires (see below).                                           │
└───────────────────────────────┬──────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 8. DEVICE B — the recipient                                          │
│    • app closed       → the OS shows the notification                │
│    • app in foreground → the app shows it locally (the remote one    │
│                          is suppressed, or it would appear twice)    │
└──────────────────────────────────────────────────────────────────────┘
```

## Two delivery paths, on purpose

| Recipient's app is… | Who generates it | Who displays it |
|---|---|---|
| **In the foreground** | The app itself, listening to the database in real time | The app (local notification) |
| **Closed / background** | The server, via the push channel | The operating system |

The foreground case is genuinely peer-to-peer-ish: the device listens to the
database directly, and no server of ours is involved. It just fails to cover the
case that matters most — a closed app.

## Design consequences

| Consequence | Reason |
|---|---|
| The client sends **IDs, never content** | The server re-reads the record, so a modified client cannot invent a message. See [05](05-security.md). |
| The intent travels on the **existing write outbox** | Durability, retries and replay come for free instead of being rebuilt. See [04](04-stateless.md). |
| Deduplication is **atomic on the database**, not in memory | Multiple instances may handle the same event; an in-memory cache cannot coordinate. |
| The push carries a **TTL** | A push channel is a queue. Without an expiry, notifications arrive days late when the device reconnects. |
| Remote pushes are **suppressed in the foreground** | The local notification already covers it. This is the least obvious rule in the system — and the one most likely to be "fixed" into a bug. |

→ Next: [04 — Going stateless](04-stateless.md)
