---
title: Unit of Work Pattern
type: pattern
created: 2026-08-05
tags: [ddd, persistence, transactions, effect]
effect-source: 80-resources/effect-source
---

# Unit of Work Pattern

## Intent

Track all changes to domain objects within a business operation and flush them to the database in a **single transaction**. Callers don't manage transactions directly — the Unit of Work does.

The pattern exists to solve one problem: **cross-aggregate consistency when domain events aren't enough**.

---

## When you need it

True DDD says: prefer [[ddd-domain-events]] for cross-aggregate consistency (eventual consistency, no distributed transaction). But sometimes you genuinely need atomicity across two aggregates — e.g. transferring funds between two `Account` aggregates where partial failure is unacceptable.

That's when Unit of Work applies.

> If you find yourself reaching for UoW frequently, it's often a sign your aggregate boundaries are wrong — reconsider whether the two aggregates should be one.

---

## In Effect: `SqlClient.withTransaction`

Effect's SQL clients expose `withTransaction` — any Effects run inside share a single DB connection and transaction. Commit on success, rollback on any failure.

```typescript
import { Effect } from "effect"
import { SqlClient } from "effect/unstable/sql"

// Any repo operations inside withTransaction share one transaction
const transferFunds = (
  fromId: AccountId,
  toId: AccountId,
  amount: Money
): Effect.Effect<void, InsufficientFunds | PersistenceError, SqlClient | AccountRepository> =>
  Effect.gen(function* () {
    const sql = yield* SqlClient
    const repo = yield* AccountRepository

    yield* sql.withTransaction(
      Effect.gen(function* () {
        const from = yield* repo.findById(fromId)
        const to = yield* repo.findById(toId)

        yield* from.debit(amount)   // domain method — may fail with InsufficientFunds
        yield* to.credit(amount)

        yield* repo.save(from)
        yield* repo.save(to)
        // both saves committed atomically, or both rolled back
      })
    )
  })
```

`withTransaction` handles nested transactions via **savepoints** automatically — you can safely nest `withTransaction` calls without double-committing.

---

## Structuring it: explicit UoW service

For larger apps, wrap `withTransaction` in an explicit service so application services don't depend on `SqlClient` directly:

```typescript
import { Effect, Context, Layer } from "effect"
import { SqlClient } from "effect/unstable/sql"

// --- Port (application layer) ---

export interface UnitOfWork {
  readonly transaction: <A, E, R>(
    effect: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E, R>
}

export class UnitOfWork extends Context.Service<UnitOfWork>()("UnitOfWork") {}

// --- Implementation (infrastructure layer) ---

export const UnitOfWorkLive = Layer.effect(
  UnitOfWork,
  Effect.gen(function* () {
    const sql = yield* SqlClient
    return UnitOfWork.of({
      transaction: (effect) => sql.withTransaction(effect),
    })
  })
)
```

Application service uses `UnitOfWork`, not `SqlClient`:

```typescript
import { Effect } from "effect"
import { UnitOfWork } from "../infrastructure/UnitOfWork"
import { AccountRepository } from "../domain/account/AccountRepository"

export const transferFunds = (
  fromId: AccountId,
  toId: AccountId,
  amount: Money
) =>
  Effect.gen(function* () {
    const uow = yield* UnitOfWork
    const repo = yield* AccountRepository

    yield* uow.transaction(
      Effect.gen(function* () {
        const from = yield* repo.findById(fromId)
        const to = yield* repo.findById(toId)
        yield* from.debit(amount)
        yield* to.credit(amount)
        yield* repo.save(from)
        yield* repo.save(to)
      })
    )
  })
```

---

## Dispatching domain events after the transaction

Events should be dispatched **after** the transaction commits — not inside it. If the event bus call fails, the data is already persisted (use the [[ddd-domain-events#transactional-outbox|transactional outbox]] for guaranteed delivery).

```typescript
export const transferFunds = (fromId: AccountId, toId: AccountId, amount: Money) =>
  Effect.gen(function* () {
    const uow = yield* UnitOfWork
    const repo = yield* AccountRepository
    const bus = yield* EventBus

    // 1. Transactional work
    const events = yield* uow.transaction(
      Effect.gen(function* () {
        const from = yield* repo.findById(fromId)
        const to = yield* repo.findById(toId)
        yield* from.debit(amount)
        yield* to.credit(amount)
        yield* repo.save(from)
        yield* repo.save(to)
        return [...from.pullEvents(), ...to.pullEvents()]
      })
    )

    // 2. Publish after commit
    yield* Effect.forEach(events, bus.publish, { discard: true })
  })
```

---

## What UoW is NOT

| Misconception | Reality |
|---|---|
| A replacement for aggregate design | If you need UoW everywhere, aggregate boundaries are wrong |
| A place for business logic | Zero logic — only transaction coordination |
| Required for every operation | Single-aggregate operations go straight through the repository |
| A way to batch arbitrary queries | It's a consistency boundary, not a performance optimization |

---

## Related

- [[ddd-repository-pattern]] — repositories participate in the UoW transaction
- [[ddd-domain-events]] — preferred alternative; events after transaction commit
- [[ddd-aggregate-root]] — transaction boundaries follow aggregate boundaries
- [[domain-driven-design]]

## References

- Fowler, *Patterns of Enterprise Application Architecture* — Unit of Work
- Evans, *DDD* ch. 6 — aggregate boundaries and transactions
- Effect source: `80-resources/effect-source/packages/effect/src/unstable/sql/SqlClient.ts`
