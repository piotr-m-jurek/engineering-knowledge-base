---
title: When Not To Use — Architecture Overkill Guide
type: concept
created: 2026-08-11
tags: [architecture, event-sourcing, event-driven, cqrs, durable-execution, queues, decision]
---

# When Not To Use — Architecture Overkill Guide

The patterns below are genuinely powerful. They are also genuinely expensive. The default question should not be "should I use event sourcing here?" but "what problem do I actually have, and is this the cheapest fix?"

Each section: what the pattern costs, the signals that mean you don't need it, and what to use instead.

---

## Event Sourcing

### What it costs

- Every read requires a projection or a snapshot. No `SELECT * FROM budgets WHERE id = ?`.
- Schema evolution is hard: old events must be readable forever, or you need upcasters.
- Debugging is non-obvious: current state is not stored anywhere directly.
- Your team needs to understand fold/replay mental model — not everyone does on first contact.
- Testing projections and read models is a separate concern from testing the write model.

### You don't need it when

**You don't care about history.** A `sessions` table, a `password_reset_tokens` table, a cache. These exist to support the current moment. Past state is irrelevant.

**Your "audit log" requirement is thin.** "Who changed this field last" is solved by adding `updated_by` and `updated_at` columns. "Show me the last 5 changes" is solved by a simple append-only `_history` table with a trigger or application-level write. You do not need full event sourcing for this.

**Your domain is CRUD.** An admin panel that manages configuration records. A CMS. A user profile editor. There are no meaningful domain events — just "someone edited this". The event log would be `RecordUpdated { before: {...}, after: {...} }` which is just a verbose diff log.

**You have a small team or early-stage product.** The overhead of projections, snapshots, and event schema management is real maintenance work. If you're still figuring out what the domain even is, you'll be migrating event schemas constantly.

### Use instead

| Need | Simpler alternative |
|------|-------------------|
| Basic audit | `updated_by`, `updated_at` columns + `_history` append table |
| Richer audit | Temporal tables (Postgres `FOR SYSTEM TIME`, or manual `valid_from`/`valid_to`) |
| Full history for one aggregate | Append-only table + current-state view |
| Time-travel queries | Temporal tables or a dedicated audit log service |

---

## Event-Driven Architecture (async messaging / broker)

### What it costs

- Eventual consistency: consumer hasn't processed yet, so the read after a write may be stale.
- Operational complexity: broker to run, monitor, scale, back up (Kafka, RabbitMQ, etc.).
- Debugging across services requires distributed tracing — a log line in service A and a log line in service B don't automatically correlate.
- At-least-once delivery means consumers must be idempotent — easy to get wrong silently.
- Schema evolution of event payloads requires coordination (or a schema registry).

### You don't need it when

**You have one service.** If `OrderService` and `InventoryService` live in the same process, a direct function call is fine. EDA is for decoupling across process/deployment boundaries. Intra-process pub/sub is usually just indirection with extra steps.

**The operation must be synchronous for UX reasons.** User submits a form and expects to see the result immediately. If you async the write and return "it'll be processed shortly", users will hit F5 and file a bug report. Sync is fine.

**Your "integration" is two services you own and deploy together.** If you always deploy them together and they always change together, they are not independently deployable microservices — they are a distributed monolith with a queue in the middle. Merge them.

**The failure mode of async is worse than the failure mode of sync.** Some operations need to fail loudly and immediately if a downstream step fails — not silently queue and fail later. Payment processing is often like this: you want to know the card was declined *now*, not in 30 seconds when the async consumer tries.

### Use instead

| Need | Simpler alternative |
|------|-------------------|
| Cross-module notification in a monolith | In-process event bus (or just a function call) |
| Decoupling two services you own | Direct HTTP/RPC call with a retry policy |
| Reliable async in one service | DB-backed job queue (pg-boss, Solid Queue, etc.) |
| Loose coupling with one consumer | Webhook (push HTTP) — no broker needed |

---

## CQRS (separate read and write models)

### What it costs

- Two models to maintain: write model (aggregates) and read model (projections).
- Read model is eventually consistent with the write model — lag exists.
- Any bug in projection logic produces silently wrong reads.
- More code: projection handlers, read repositories, sync/rebuild tooling.

### You don't need it when

**Your read and write shapes are the same.** If you write a `User` and read back a `User` with the same fields, CQRS adds a projection for zero benefit. A single repository with `findById` and `save` is correct.

**Your query load is low.** CQRS is often justified by "reads are 10x more than writes, separate them for scale." If you have 100 users, this is premature optimization.

**Your queries are simple.** If every query is "fetch aggregate by ID", a normal repository handles it. CQRS projections shine for cross-aggregate queries ("all orders over $500 by users in Germany") — if you don't have those, you don't have the problem.

**You're in early product iteration.** The read model needs rebuilding every time a projection changes. If your domain model is still in flux, you'll rebuild constantly.

### Use instead

| Need | Simpler alternative |
|------|-------------------|
| Better read performance | DB indexes, a read replica, or a materialized view |
| Cross-aggregate query | A JOIN or a DB view — not a separate model |
| Separate scaling of reads/writes | Read replica (one line of config in most hosted DBs) |

---

## Durable Execution (Temporal, Effect Workflows)

### What it costs

- Workflow code must be deterministic — no `Date.now()`, no `Math.random()`, no direct IO.
- Determinism bugs are silent and only show up on replay, which may be rare.
- Operational overhead: Temporal Server to run (or pay for Temporal Cloud).
- Debugging is in Temporal UI / workflow history, not your normal logs.
- TypeScript SDK adds startup latency (workflow sandboxing).
- Versioning in-flight workflows when you change code is non-trivial.

### You don't need it when

**The process completes in one request.** If the whole thing runs in under 30 seconds and doesn't wait for external humans or async external systems, a normal async function with retries is fine.

**The steps are all in your DB transaction.** Place order + reserve inventory + record payment all in one transaction = all-or-nothing for free. No workflow engine needed.

**"Retry on failure" is the full requirement.** If you just need "call this HTTP endpoint, retry 3 times on failure", a job queue with retry policy (pg-boss, Sidekiq, etc.) is the right tool. Durable execution is for processes that span hours/days or require human-in-the-loop waits.

**You're a solo developer on a weekend project.** The operational cost of Temporal doesn't pay off at small scale. A cron job + status column in your DB is fine.

### Use instead

| Need | Simpler alternative |
|------|-------------------|
| Retry failed HTTP calls | HTTP client with retry policy (exponential backoff) |
| Background jobs with retry | pg-boss, Sidekiq, BullMQ, Solid Queue |
| Multi-step process, all in-process | Plain `async/await` with error handling |
| Long wait (hours) for one event | DB row with `status` + cron + webhook handler |
| Process manager pattern | Saga / process manager aggregate (see [[process-manager-saga]]) |

---

## Async Queues (in general)

### What it costs

- Visibility: a job in a queue is invisible until it fails or completes. Debugging requires queue tooling.
- Order is not guaranteed in most brokers under load.
- Idempotency burden: consumers must handle duplicate delivery.
- Backpressure can mask problems: queue depth growing = something is wrong, but it's silent.

### You don't need it when

**The work takes under a second and the user is waiting.** Just do it synchronously. The queue adds latency, complexity, and a failure mode (queue full / consumer dead) for no gain.

**You have one consumer and it's in the same process.** An in-process channel or a simple function call is simpler and faster with no broker.

**Your "queue" is one table.** A `jobs` table polled by a background thread *is* a queue. If that's all you need, don't add Kafka.

### Use instead

| Need | Simpler alternative |
|------|-------------------|
| Offload slow work from request path | Respond immediately, do work in a goroutine/thread, no broker |
| Scheduled work | Cron job |
| Simple background jobs | DB-backed queue (pg-boss, etc.) — same DB, no new infra |
| Rate limiting downstream calls | Semaphore / token bucket in-process |

---

## The general heuristic

> **Use the simplest thing that handles your actual failure modes.**

Identify the failure mode first:
- "We lose history" → event sourcing (or simpler audit log)
- "Services are coupled" → EDA (or just better API design)
- "Reads are slow" → index/replica/materialized view (or CQRS)
- "Long-running process crashes" → durable execution (or job table + cron)
- "Request path is slow" → async queue (or just faster code)

Most of the time, the failure mode doesn't exist yet. Add complexity when you feel the pain, not in anticipation of it.

---

## Related

- [[event-sourcing]]
- [[durable-execution]]
- [[ddd-domain-events]]
- [[ports-and-adapters]]
- [[service-coupling]] — deep dive on "services are coupled" → EDA decision
- [[architecture-decision-records]] — when you do choose one of these patterns, write an ADR
