# System overview — phase 1 → phase 2

One change defines the whole story: **the state leaves the server.**

## Phase 1 — one process, always on, state in memory

```mermaid
flowchart TB
  C["Client — PWA / native<br/>auth · direct data access · local notifications"]
  DB[("Firebase RTDB<br/>SINGLE SOURCE OF TRUTH<br/>trips · expenses · stops · members")]
  S["Node — ONE process, always on<br/>/api/ai · /api/jobs/scan · /api/places/*<br/>quotas · rate limit · telemetry<br/>RTDB LISTENER — group push<br/>also serves the static site"]
  AI["AI provider"]
  P["Push channel"]
  H["Render free tier · region Oregon<br/>Cloudflare edge in front"]

  C -->|"realtime (WebSocket)"| DB
  C -->|"HTTPS + ID token — side effects only"| S
  S -->|"re-reads records"| DB
  S --> AI
  S --> P
  P --> C
  S --- H
```

The listener holds **six pieces of state in memory** and needs the process to
stay alive — so scale-to-zero is impossible by construction, and the service
sleeps on the free tier anyway. See [03](../docs/03-architecture-phase-1.md).

## Phase 2 — stateless, client-driven, scales to zero

```mermaid
flowchart TB
  C["Client — native (PWA in dev only)<br/>auth · direct data access<br/>durable outbox · coalescing · local notifications"]
  DB[("Firebase RTDB<br/>SINGLE SOURCE OF TRUTH<br/>+ atomic dedup markers")]
  S["Server — STATELESS<br/>verify ID token · re-read records<br/>quota · build + send push<br/>enqueue long jobs<br/>nothing important in memory"]
  AI["AI provider"]
  P["Push channel"]
  Q["Cloud Tasks — durable jobs"]
  H["Cloud Run · EU · min=0 + CPU boost<br/>Secret Manager · Cloudflare + Pages"]

  C -->|"realtime (WebSocket)"| DB
  C -->|"POST /api/notify + ID token"| S
  S -->|"re-read + resolve recipients"| DB
  S --> AI
  S --> P
  S --> Q
  P --> C
  S --- H
```

The trigger moved to the client that performed the write, and the intent rides
the **outbox that already existed** — so durability, retry and replay came for
free. See [04](../docs/04-architecture-phase-2.md).

## What actually changed

| | Phase 1 | Phase 2 |
|---|---|---|
| Detects the change | a server process watching the database | the client that performed the write |
| Deduplication | in-memory cache | atomic transaction on the database |
| Coalescing | in-memory batch window | in the client, before sending |
| Scheduling | a timer in a live process | a durable on-disk outbox |
| Scaling | one instance, always on | many instances, scale to zero |
| Hosting bill at idle | free tier with a sleeping process | **€0** |

The dividing rule never changed: **data goes direct from the client** (authorised
by versioned rules), **side effects go through the server** (push, model calls,
quotas, privileges).
