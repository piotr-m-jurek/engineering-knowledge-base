---
title: Event Sourcing
type: concept
created: 2026-08-11
tags: [distributed-systems, events, persistence, cqrs, ddd, architecture]
---

# Event Sourcing

## The problem

You save the current state of an object to a database. Later it's wrong. You want to know: how did it get this way? Who changed it? What was it last Tuesday?

You can't answer those questions. The write overwrote the history. You only have the present snapshot.

The standard fix is an audit log column or a separate audit table — but it's bolted on, optional, inconsistently maintained, and still doesn't let you reconstruct intermediate states.

---

## Core idea

> Don't store current state. Store the sequence of events that produced it. Derive current state by replaying them.

Instead of a `budgets` table with one row per budget showing the current values:

| id | income_cents | updated_at |
|----|-------------|------------|
| abc123 | 300000 | 2026-08-11 |

You store a `budget_events` table with one row per thing that happened:

| seq | budget_id | event                                      | occurred_at      |
| --- | --------- | ------------------------------------------ | ---------------- |
| 1   | abc123    | `BudgetCreated { income: 280000 }`         | 2026-07-01 09:00 |
| 2   | abc123    | `EnvelopeCreated { name: "Rent", ... }`    | 2026-07-01 09:01 |
| 3   | abc123    | `FundsAllocated { envelope: "Rent", ... }` | 2026-07-15 14:22 |
| 4   | abc123    | `BudgetIncomeUpdated { income: 300000 }`   | 2026-08-01 10:00 |

To get the current budget: replay events 1–4 in order. To get the budget as it was on July 31: replay events 1–3. The event log is the source of truth. The snapshot is a derived view.

---

## How replay works

A **fold** (left-reduce) over the event sequence, starting from an empty initial state:

```typescript
type BudgetEvent = BudgetCreated | EnvelopeCreated | FundsAllocated | BudgetIncomeUpdated

function applyEvent(state: BudgetState, event: BudgetEvent): BudgetState {
  switch (event._tag) {
    case "BudgetCreated":
      return { ...state, id: event.budgetId, income: event.income, envelopes: [] }
    case "EnvelopeCreated":
      return { ...state, envelopes: [...state.envelopes, { id: event.envelopeId, name: event.name, allocation: event.allocation }] }
    case "FundsAllocated":
      return { ...state, envelopes: state.envelopes.map(e =>
        e.id === event.envelopeId ? { ...e, allocation: event.newAllocation } : e
      )}
    case "BudgetIncomeUpdated":
      return { ...state, income: event.income }
  }
}

// Replay: fold all events from the beginning
const currentState = events.reduce(applyEvent, emptyBudgetState)
```

`applyEvent` is a pure function — no I/O, no side effects. This is also why testing is easy: feed it a list of events, assert the resulting state.

---

## What you get

**Complete audit trail for free.** The event log is the audit log. Every change is recorded with who made it and when. Nothing to bolt on.

**Point-in-time reconstruction.** Replay events up to any timestamp → state at that moment. "What did the budget look like on the 15th?" is a query, not an impossibility.

**Temporal queries.** "Show me all budgets that had their income changed in July." Filter the event log.

**Event replay for debugging.** Reproduce any bug by replaying the exact sequence of events that led to it.

**Natural fit for [[ddd-domain-events]].** Aggregates already produce events (`pullEvents()`). Persisting those events instead of derived state is a short step.

**Enables CQRS.** The write side appends events. The read side projects events into whatever shape queries need — flat tables, denormalized views, search indexes. Read and write models are independent.

---

## The tradeoff

**Current state is O(n) to compute.** Replaying 10,000 events to answer "what's the balance?" is slow. The fix is **snapshots**: periodically checkpoint the current state so replay starts from the snapshot, not event 1.

```
Snapshot at event 950 → replay events 951–1000 → current state
```

Snapshots are an optimisation. They don't change the event log — it remains the source of truth.

**Query patterns change.** You can't `SELECT * FROM budgets WHERE income > 200000`. You either project a read model (CQRS) or accept scanning events. Most production systems project into a separate read database.

**Schema evolution is hard.** Old events are immutable. If the shape of `EnvelopeCreated` changes, you must handle both versions in `applyEvent`. Upcasting (transforming old format to new on load) is the standard technique.

**Not suitable everywhere.** A user's profile doesn't benefit from event sourcing. An `envelopes` table where you write the current balance directly is fine. Use event sourcing when history matters, or when CQRS is needed, or when the domain naturally produces events.

---

## Event sourcing vs event-driven

These are different things that sound the same.

**Event-driven architecture** — services communicate by publishing and subscribing to events over a bus. The *transport* is events. State is still stored as snapshots in each service's own database.

**Event sourcing** — state is stored as events in a log. The *persistence model* is events. The architecture may or may not be event-driven.

A system can be one, both, or neither. Kafka + a Postgres table with `current_balance` is event-driven but not event-sourced. An event-sourced aggregate with an in-memory `PubSub` bus is event-sourced but arguably only internally event-driven.

---

## Relationship to durable execution

[[durable-execution]] uses the same core idea for a different purpose.

In event sourcing, the log records *what happened to domain state*. On restart, replay the log → reconstruct current domain state.

In durable execution (Temporal, Effect Workflows), the log records *what steps the workflow executed*. On restart, replay the log → reconstruct where the workflow was up to, then continue from there.

Same mechanism. Different subject being reconstructed. This is not a coincidence — Temporal's architecture was directly influenced by event sourcing thinking.

```
Event sourcing:        events → domain state
Durable execution:     events → workflow execution position
```

---

## In the DDD finance app

The finance app doesn't use event sourcing for persistence by default — it uses a standard repository that stores current state. But the domain is structured to make adding it straightforward:

- Aggregates already collect events in `_events` and expose them via `pullEvents()`
- Those events (`BudgetCreated`, `EnvelopeCreated`, `FundsAllocated`, ...) are exactly what an event store would persist
- `applyEvent` would be the inverse of the aggregate's mutating methods — already implicit in how the aggregate builds its state

Swapping `InMemoryBudgetRepository` for an `EventSourcedBudgetRepository` would:
1. On `save`: append `budget.pullEvents()` to the event store
2. On `findById`: load events for that ID, fold with `applyEvent`, return reconstituted aggregate

Same interface. Same domain code. Different `Layer`.

---

## Related

- [[ddd-domain-events]] — aggregates produce events; event sourcing persists them as the primary record
- [[durable-execution]] — uses the same replay mechanism for workflow state rather than domain state
- [[ddd-aggregate-root]] — the aggregate whose events are stored
- [[ddd-repository-pattern]] — the port; event-sourced repo is one implementation
- [[process-manager-saga]] — process managers are natural candidates for event sourcing (their state IS their event history)
- [[ddd-finance-app]] — the reference project; events already produced, storage model is swappable

## References

- Evans, *DDD* — domain events as first-class records
- Vernon, *IDDD* ch. 8 — event sourcing implementation with aggregates
- Greg Young, *CQRS and Event Sourcing* (talk, 2010) — canonical introduction
- Martin Fowler, *Event Sourcing* — https://martinfowler.com/eaaDev/EventSourcing.html
