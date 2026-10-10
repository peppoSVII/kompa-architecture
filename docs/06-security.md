# 06 — Security model: a write path that trusts nobody

## The problem being solved

There is an endpoint that takes a request from a mobile client and causes
notifications to be delivered to **other people's phones**. That is a powerful
capability, and the client making the request is a binary that anyone can
download, decompile and modify.

So the design assumption is: **assume the client is hostile.**

## The core decision: no pass-through

The client is allowed to send exactly this:

```json
{ "gid": "group123", "kind": "expense",
  "items": [ { "id": "e42", "op": "added" } ] }
```

No text. No amounts. No names. No recipients.

And the server **never forwards the client's bytes** to the push channel. It
reads the record from the database, builds the message itself from a pure
template, and sends that.

```
client → { id }  ──►  SERVER  ──►  reads the record from the database
                                 ──►  builds the text from a template
                                 ──►  sends to the recipients it resolved
```

There is no code path in which something the client wrote reaches another
device. That single property removes an entire class of attack — injected
content, phishing text, spoofed amounts — rather than defending against it.

## The defences, in order

| # | Defence | What it prevents |
|---|---|---|
| 1 | **ID token verification** — the actor is derived from the token, never the payload | Impersonating another user |
| 2 | **Event type → allow-list of database paths** | Arbitrary path access |
| 3 | **Item count cap** (default 50) | "Notify a hundred thousand records" abuse |
| 4 | **Normalisation** of ids and operation types | Value injection |
| 5 | **Membership check** → 403 | Notifying groups you do not belong to |
| 6 | **Rate limiting** → 429 | Spam |
| 7 | **Records re-read from the database** | Inventing content |
| 8 | **Message text built by the server** (pure, unit-tested module) | Injecting titles, links, markup |
| 9 | **Actor name resolved from the database** | Claiming to be someone else |
| 10 | **Recipients resolved from the database** | Choosing who gets notified |
| 11 | **Atomic dedup transaction** | Spam even from a buggy client |
| 12 | **Request body size cap** → 413 | Memory exhaustion |

## Threat model: what can a modified client actually do?

Assume the attacker has the app, a **valid account**, and full control of the
client code. Then:

| Attempt | Outcome |
|---|---|
| Notify a group they are not a member of | ❌ 403 |
| Invent the text ("Mario added €900") | ❌ the server re-reads the record |
| Choose the recipients | ❌ resolved from the database |
| Impersonate another user | ❌ the actor comes from a signed token |
| Notify a record that does not exist | ❌ 409 |
| Spam | ❌ rate limit + atomic dedup |
| Send a malicious payload | ❌ there is no pass-through |
| **Worst case** | 🟡 **an unpleasant notification to their own group** |

That last row is the point. The blast radius is confined to a group the attacker
already belongs to — and they could achieve the same annoyance by simply adding
a real expense. There is no privilege escalation, no cross-tenant access, and no
delivery to a device the attacker cannot already reach.

## Why a "virus" is not deliverable through this channel

Three independent reasons:

| Reason | Detail |
|---|---|
| **A notification does not execute code** | It is text plus string key/value pairs. The OS renders it and stops. |
| **The payload is built by the server** | Fixed keys, server-chosen values. The client does not choose the shape. |
| **On tap, the app only navigates** | A route plus a validated id. No content is interpreted. |

## Defences that deliberately are *not* used here

| Defence | Why it is irrelevant |
|---|---|
| **CORS** | Protects a browser, not a native client. It matters only for the web build. |
| **Database security rules** | The admin SDK bypasses them. They are the **client's** defence, not the server's. |

This distinction matters: it is easy to look at a well-written rules file and
assume the server is covered. It is not — the server runs with full privileges
by design, which is exactly why every check above has to exist in code.

## Known gaps — not closed yet

Listed rather than hidden:

| Gap | Impact | Status |
|---|---|---|
| **App attestation is not implemented** | Anyone with an account can call the endpoint directly with a script. The twelve defences above still hold, but there is no proof the request came from the genuine app binary. | Designed, not built |
| **The rate limit is an in-memory map** | With N instances the effective limit becomes the limit × N. | On the list before the scale-to-zero migration |
| **Push delivery works on one platform only** | The client registers the platform's raw device token. That token format is accepted by the push API on Android but not on iOS, where a different token type is required. iOS currently gets only in-app notifications. | Known, fix requires a provider-side key plus a client change |
| **API keys are present in the client bundle** | ⚠️ Expected, not a leak. Web API keys are **public identifiers** for the project, not secrets; authorisation comes from security rules and server-side checks. Documented so it is not mistaken for a finding. | By design |

## The principle

> **Never trust the client.** It may report *what changed* — an id.
> It never gets to decide *how it is described*.
>
> Everything that is text, recipients, identity or quantity is reconstructed
> server-side from the database.

→ Next: [07 — Cost model](07-cost.md)
