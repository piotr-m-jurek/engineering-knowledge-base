---
title: DDD Finance App — Build Walkthrough
type: concept
created: 2026-08-10
tags: [ddd, effect, walkthrough, finance, project]
---

# DDD Finance App — Build Walkthrough

A step-by-step explanation of building the envelope budgeting app, with commentary on how DDD layers interact at each stage.

> See [[domain-driven-design]] for foundational concepts. See the weekend plan summary for scope and package structure.

---

## How the layers relate — the mental model

Before touching code, internalize this flow. Every decision below maps to it.

```
Request arrives
      │
      ▼
┌─────────────────────────┐
│   Presentation / API    │  HTTP routes — validates HTTP input, calls application service
└────────────┬────────────┘
             │ calls
             ▼
┌─────────────────────────┐
│   Application Service   │  Orchestrates: load → call domain → save → publish events
│   (no business logic)   │  Knows about repos and event bus. Knows nothing about HTTP.
└────────────┬────────────┘
             │ calls
             ▼
┌─────────────────────────┐
│      Domain Layer       │  Aggregates, value objects, domain services.
│   (pure, no I/O)        │  Returns new state + domain events. Fails with domain errors.
└────────────┬────────────┘
             │ interfaces defined here, implemented below
             ▼
┌─────────────────────────┐
│   Infrastructure Layer  │  Repository impls, event bus impl, DB, HTTP clients.
│   (Layer in Effect)     │  Wired together at app startup via Layer.mergeAll.
└─────────────────────────┘
```

**Key rule:** dependencies point downward only. Domain knows nothing about infrastructure. Infrastructure knows nothing about HTTP. Application service is the only place that knows about both domain and infrastructure interfaces.

---

## Step 1 — Shared Kernel

**Layer: Domain (shared)**

The shared kernel is a tiny, stable package imported by every bounded context. It defines concepts that are genuinely shared — not because it's convenient, but because they'd have the exact same meaning and rules everywhere.

```
packages/shared-kernel/src/
  Money.ts          ← integer cents, never floats
  BudgetPeriod.ts   ← { year, month }, immutable
  EntityId.ts       ← branded ID factory
  DomainEvent.ts    ← base interface all events implement
```

### Money

`Money` is a value object. Two `Money` instances with the same cents and currency are equal — identity doesn't matter.

```typescript
// packages/shared-kernel/src/Money.ts
import { Data, Effect, Brand } from "effect"

class InvalidMoneyAmount extends Data.TaggedError("InvalidMoneyAmount")<{
  readonly cents: number
}> {}

export class Money extends Data.Class<{
  readonly cents: number    // always integer, always in minor units (cents)
  readonly currency: "USD"  // extend later
}> {
  // Smart constructor — domain rule lives here, not in callers
  static make(cents: number): Effect.Effect<Money, InvalidMoneyAmount> {
    if (!Number.isInteger(cents))
      return Effect.fail(new InvalidMoneyAmount({ cents }))
    return Effect.succeed(new Money({ cents, currency: "USD" }))
  }

  add(other: Money): Money {
    return new Money({ cents: this.cents + other.cents, currency: "USD" })
  }

  subtract(other: Money): Money {
    return new Money({ cents: this.cents - other.cents, currency: "USD" })
  }

  isNegative(): boolean { return this.cents < 0 }
  isZero(): boolean { return this.cents === 0 }
}
```

**Why `Data.Class`?** Structural equality for free. `Equal.equals(new Money({cents:100, currency:"USD"}), new Money({cents:100, currency:"USD"}))` returns `true`. Without `Data.Class` that would be reference equality — two separate instances would never be equal.

**Why integer cents?** Floating-point arithmetic breaks money. `0.1 + 0.2 !== 0.3` in JavaScript. Store everything in cents, display by dividing by 100.

**Why a smart constructor returning `Effect`?** Domain rules live in one place. Every caller that constructs `Money` goes through the same validation. If the rule changes, it changes once.

### BudgetPeriod

```typescript
// packages/shared-kernel/src/BudgetPeriod.ts
import { Data, Effect } from "effect"

class InvalidBudgetPeriod extends Data.TaggedError("InvalidBudgetPeriod")<{
  readonly year: number
  readonly month: number
}> {}

export class BudgetPeriod extends Data.Class<{
  readonly year: number   // e.g. 2026
  readonly month: number  // 1–12
}> {
  static make(year: number, month: number): Effect.Effect<BudgetPeriod, InvalidBudgetPeriod> {
    if (month < 1 || month > 12 || !Number.isInteger(year))
      return Effect.fail(new InvalidBudgetPeriod({ year, month }))
    return Effect.succeed(new BudgetPeriod({ year, month }))
  }

  next(): BudgetPeriod {
    return this.month === 12
      ? new BudgetPeriod({ year: this.year + 1, month: 1 })
      : new BudgetPeriod({ year: this.year, month: this.month + 1 })
  }

  label(): string { return `${this.year}-${String(this.month).padStart(2, "0")}` }
}
```

### EntityId factory

```typescript
// packages/shared-kernel/src/EntityId.ts
import { Brand } from "effect"

// Call this once per entity type to get a branded ID
export const makeIdBrand = <Tag extends string>() => Brand.nominal<string & Brand.Brand<Tag>>()
```

Usage in each context:
```typescript
// budgeting/src/domain/ids.ts
import { makeIdBrand } from "shared-kernel"

export type BudgetId = string & Brand.Brand<"BudgetId">
export const BudgetId = makeIdBrand<"BudgetId">()

export type EnvelopeId = string & Brand.Brand<"EnvelopeId">
export const EnvelopeId = makeIdBrand<"EnvelopeId">()
```

**Why branded IDs?** `repo.findById(userId)` won't accidentally accept an `orderId` — TypeScript rejects the wrong brand at compile time, no runtime check needed.

---

## Step 2 — Budgeting Domain Layer

**Layer: Domain**

This is the heart of the app. Pure TypeScript — no database, no HTTP, no Effect requirements in `R`. Just data structures and rules.

```
packages/budgeting/src/domain/
  ids.ts         ← BudgetId, EnvelopeId
  errors.ts      ← all domain errors for this context
  events.ts      ← all domain events for this context
  Envelope.ts    ← child entity
  Budget.ts      ← aggregate root
```

### Domain errors

```typescript
// packages/budgeting/src/domain/errors.ts
import { Data } from "effect"
import { BudgetId, EnvelopeId } from "./ids"
import { Money, BudgetPeriod } from "shared-kernel"

export class AllocationExceedsBudgetIncome extends Data.TaggedError("AllocationExceedsBudgetIncome")<{
  readonly requested: Money
  readonly available: Money
}> {}

export class DuplicateEnvelopeName extends Data.TaggedError("DuplicateEnvelopeName")<{
  readonly name: string
}> {}

export class EnvelopeNotFound extends Data.TaggedError("EnvelopeNotFound")<{
  readonly id: EnvelopeId
}> {}

export class BudgetNotFound extends Data.TaggedError("BudgetNotFound")<{
  readonly id: BudgetId
}> {}

export class BudgetAlreadyExists extends Data.TaggedError("BudgetAlreadyExists")<{
  readonly period: BudgetPeriod
}> {}
```

**Why `Data.TaggedError`?** Three reasons:
1. `_tag` discriminant enables `Effect.catchTag("AllocationExceedsBudgetIncome", ...)` — exhaustive handling
2. Structural equality — two errors with the same fields are equal (useful in tests)
3. They're `yield*`-able in `Effect.gen` — no need to wrap in `Effect.fail`

### Domain events

```typescript
// packages/budgeting/src/domain/events.ts
import { Data } from "effect"
import { BudgetId, EnvelopeId } from "./ids"
import { Money, BudgetPeriod } from "shared-kernel"

export class BudgetCreated extends Data.TaggedClass("BudgetCreated")<{
  readonly budgetId: BudgetId
  readonly period: BudgetPeriod
  readonly income: Money
  readonly occurredAt: Date
}> {}

export class EnvelopeCreated extends Data.TaggedClass("EnvelopeCreated")<{
  readonly budgetId: BudgetId
  readonly envelopeId: EnvelopeId
  readonly name: string
  readonly allocation: Money
  readonly occurredAt: Date
}> {}

export class FundsAllocated extends Data.TaggedClass("FundsAllocated")<{
  readonly budgetId: BudgetId
  readonly envelopeId: EnvelopeId
  readonly newAllocation: Money
  readonly occurredAt: Date
}> {}

export type BudgetingEvent = BudgetCreated | EnvelopeCreated | FundsAllocated
```

**Why `Data.TaggedClass` not `Data.TaggedError`?** Events are data, not errors. They don't fail, they're just records of what happened.

### Envelope entity

```typescript
// packages/budgeting/src/domain/Envelope.ts
import { Data } from "effect"
import { EnvelopeId } from "./ids"
import { Money } from "shared-kernel"

// Internal to Budget aggregate — never referenced from outside
export class Envelope extends Data.Class<{
  readonly id: EnvelopeId
  readonly name: string
  readonly allocation: Money  // how much was budgeted
  readonly balance: Money     // allocation minus spent (updated when transactions arrive)
}> {
  withAllocation(allocation: Money): Envelope {
    return new Envelope({ ...this, allocation })
  }

  withBalance(balance: Money): Envelope {
    return new Envelope({ ...this, balance })
  }

  isOverspent(): boolean {
    return this.balance.isNegative()
  }
}
```

`Envelope` is an **entity** — it has identity (`EnvelopeId`) and can change over time. But it's a child entity: external code holds `BudgetId`, not `EnvelopeId`. You get to envelopes through `Budget`.

### Budget aggregate root

```typescript
// packages/budgeting/src/domain/Budget.ts
import { Data, Effect, HashMap, Option } from "effect"
import { crypto } from "node:crypto"
import { BudgetId, EnvelopeId } from "./ids"
import { Money, BudgetPeriod } from "shared-kernel"
import { Envelope } from "./Envelope"
import {
  AllocationExceedsBudgetIncome,
  DuplicateEnvelopeName,
  EnvelopeNotFound,
} from "./errors"
import { BudgetingEvent, EnvelopeCreated, FundsAllocated, BudgetCreated } from "./events"

export class Budget {
  // Collected but unpublished events — cleared by pullEvents()
  private _events: BudgetingEvent[] = []

  constructor(
    readonly id: BudgetId,
    readonly period: BudgetPeriod,
    readonly income: Money,
    private _envelopes: HashMap.HashMap<EnvelopeId, Envelope>,
  ) {}

  // Read-only view — defensive copy so callers can't mutate internal state
  get envelopes(): HashMap.HashMap<EnvelopeId, Envelope> {
    return this._envelopes
  }

  // --- Domain methods ---

  // Invariant: envelope names unique within this budget
  // Invariant: total allocation after adding must not exceed income
  addEnvelope(
    name: string,
    allocation: Money,
  ): Effect.Effect<void, DuplicateEnvelopeName | AllocationExceedsBudgetIncome> {
    return Effect.gen(() => {
      // Check name uniqueness
      const nameTaken = HashMap.some(this._envelopes, (e) => e.name === name)
      if (nameTaken)
        return yield* Effect.fail(new DuplicateEnvelopeName({ name }))

      // Check allocation headroom
      const totalAllocated = this.totalAllocated()
      const newTotal = totalAllocated.add(allocation)
      if (newTotal.cents > this.income.cents)
        return yield* Effect.fail(new AllocationExceedsBudgetIncome({
          requested: allocation,
          available: this.income.subtract(totalAllocated),
        }))

      const id = EnvelopeId(crypto.randomUUID())
      const envelope = new Envelope({ id, name, allocation, balance: allocation })
      this._envelopes = HashMap.set(this._envelopes, id, envelope)
      this._events.push(new EnvelopeCreated({
        budgetId: this.id,
        envelopeId: id,
        name,
        allocation,
        occurredAt: new Date(),
      }))
    })
  }

  reallocate(
    envelopeId: EnvelopeId,
    newAllocation: Money,
  ): Effect.Effect<void, EnvelopeNotFound | AllocationExceedsBudgetIncome> {
    return Effect.gen(() => {
      const envelope = HashMap.get(this._envelopes, envelopeId)
      if (Option.isNone(envelope))
        return yield* Effect.fail(new EnvelopeNotFound({ id: envelopeId }))

      // Total excluding this envelope, plus the new amount
      const otherTotal = this.totalAllocated().subtract(envelope.value.allocation)
      const newTotal = otherTotal.add(newAllocation)
      if (newTotal.cents > this.income.cents)
        return yield* Effect.fail(new AllocationExceedsBudgetIncome({
          requested: newAllocation,
          available: this.income.subtract(otherTotal),
        }))

      this._envelopes = HashMap.set(
        this._envelopes,
        envelopeId,
        envelope.value.withAllocation(newAllocation),
      )
      this._events.push(new FundsAllocated({
        budgetId: this.id,
        envelopeId,
        newAllocation,
        occurredAt: new Date(),
      }))
    })
  }

  // Called from Transactions context via event (not direct call)
  applySpend(envelopeId: EnvelopeId, amount: Money): Effect.Effect<void, EnvelopeNotFound> {
    return Effect.gen(() => {
      const envelope = HashMap.get(this._envelopes, envelopeId)
      if (Option.isNone(envelope))
        return yield* Effect.fail(new EnvelopeNotFound({ id: envelopeId }))
      const updated = envelope.value.withBalance(envelope.value.balance.subtract(amount))
      this._envelopes = HashMap.set(this._envelopes, envelopeId, updated)
    })
  }

  // --- Queries ---

  totalAllocated(): Money {
    return HashMap.reduce(
      this._envelopes,
      new Money({ cents: 0, currency: "USD" }),
      (acc, e) => acc.add(e.allocation),
    )
  }

  unallocated(): Money {
    return this.income.subtract(this.totalAllocated())
  }

  // --- Event collection ---

  // Application service calls this AFTER repo.save() — not before
  pullEvents(): BudgetingEvent[] {
    const events = [...this._events]
    this._events = []
    return events
  }

  // Factory — creates a new Budget and immediately records BudgetCreated
  static create(
    period: BudgetPeriod,
    income: Money,
  ): Budget {
    const id = BudgetId(crypto.randomUUID())
    const budget = new Budget(id, period, income, HashMap.empty())
    budget._events.push(new BudgetCreated({ budgetId: id, period, income, occurredAt: new Date() }))
    return budget
  }
}
```

**What this demonstrates:**

- All invariants enforced in the aggregate root — no caller can bypass them
- `_envelopes` is private — the only way to add one is `addEnvelope()` which checks rules
- Events are collected internally and only released via `pullEvents()` — timing is controlled
- No infrastructure anywhere — no DB calls, no `fetch`, no file I/O
- `Effect` is used purely for typed errors on domain methods — `R` is always `never`

---

## Step 3 — Repository port

**Layer: Domain (interface) / Infrastructure (implementation)**

The domain layer defines what it needs from persistence as a pure interface. It does not know — and must never know — whether data lives in SQLite, Postgres, or memory.

```typescript
// packages/budgeting/src/ports/BudgetRepository.ts
import { Context, Effect } from "effect"
import { Budget } from "../domain/Budget"
import { BudgetId } from "../domain/ids"
import { BudgetNotFound, BudgetAlreadyExists } from "../domain/errors"
import { BudgetPeriod } from "shared-kernel"

// PersistenceError lives in shared-kernel or infrastructure — domain doesn't define it
import { PersistenceError } from "shared-kernel"

export interface IBudgetRepository {
  findById(id: BudgetId): Effect.Effect<Budget, BudgetNotFound | PersistenceError>
  findByPeriod(period: BudgetPeriod): Effect.Effect<Budget | null, PersistenceError>
  save(budget: Budget): Effect.Effect<void, PersistenceError>
}

// Context.Tag makes this injectable via Effect's service system
export class BudgetRepository extends Context.Tag("BudgetRepository")<
  BudgetRepository,
  IBudgetRepository
>() {}
```

**Layer interaction here:** The domain defines the shape (`IBudgetRepository`). The infrastructure layer provides the actual object. The application layer uses the tag to ask for it — `yield* BudgetRepository` — and Effect resolves it from the `Layer` at runtime.

This is the **Dependency Inversion Principle** in DDD: domain doesn't depend on infra; infra depends on domain.

### In-memory implementation

```typescript
// packages/budgeting/src/infra/InMemoryBudgetRepository.ts
import { Effect, Layer, Ref, HashMap, Option } from "effect"
import { BudgetRepository } from "../ports/BudgetRepository"
import { Budget } from "../domain/Budget"
import { BudgetId } from "../domain/ids"
import { BudgetNotFound } from "../domain/errors"
import { BudgetPeriod, PersistenceError } from "shared-kernel"

export const InMemoryBudgetRepositoryLive = Layer.effect(
  BudgetRepository,
  Effect.gen(function* () {
    // Ref is Effect's mutable reference — safe for concurrent access
    const store = yield* Ref.make(HashMap.empty<BudgetId, Budget>())

    return BudgetRepository.of({
      findById: (id) =>
        Ref.get(store).pipe(
          Effect.flatMap((map) =>
            Option.match(HashMap.get(map, id), {
              onNone: () => Effect.fail(new BudgetNotFound({ id })),
              onSome: Effect.succeed,
            })
          )
        ),

      findByPeriod: (period) =>
        Ref.get(store).pipe(
          Effect.map((map) => {
            const found = HashMap.findFirst(map, (b) =>
              Equal.equals(b.period, period)
            )
            return Option.match(found, { onNone: () => null, onSome: ([, b]) => b })
          })
        ),

      save: (budget) =>
        Ref.update(store, HashMap.set(budget.id, budget)),
    })
  })
)
```

**Layer interaction here:** `Layer.effect(BudgetRepository, ...)` says "to satisfy the `BudgetRepository` service requirement, run this effect to produce the implementation." The `Ref` is created once at layer initialization and shared across all uses.

---

## Step 4 — Application Service

**Layer: Application**

This is the orchestration layer. It answers: "what happens when a user clicks 'Create Budget'?" It knows about repositories and event bus — but contains zero business logic itself.

```typescript
// packages/budgeting/src/application/CreateBudgetService.ts
import { Effect } from "effect"
import { BudgetRepository } from "../ports/BudgetRepository"
import { Budget } from "../domain/Budget"
import { BudgetAlreadyExists } from "../domain/errors"
import { EventBus } from "shared-kernel"  // port defined in shared kernel
import { Money, BudgetPeriod } from "shared-kernel"

export interface CreateBudgetInput {
  year: number
  month: number
  incomeCents: number
}

// R type makes dependencies explicit: this function needs BudgetRepository and EventBus
export const createBudget = (
  input: CreateBudgetInput,
): Effect.Effect<Budget, BudgetAlreadyExists | InvalidMoneyAmount | InvalidBudgetPeriod, BudgetRepository | EventBus> =>
  Effect.gen(function* () {
    const repo = yield* BudgetRepository
    const bus = yield* EventBus

    // Validate value objects (domain rules fire here)
    const period = yield* BudgetPeriod.make(input.year, input.month)
    const income = yield* Money.make(input.incomeCents)

    // Check idempotency — don't create duplicate budgets for same period
    const existing = yield* repo.findByPeriod(period)
    if (existing !== null)
      return yield* Effect.fail(new BudgetAlreadyExists({ period }))

    // Call domain factory — invariants are in the domain, not here
    const budget = Budget.create(period, income)

    // Persist first, THEN publish events — order matters
    yield* repo.save(budget)
    const events = budget.pullEvents()
    yield* Effect.forEach(events, bus.publish, { discard: true })

    return budget
  })
```

**Why persist before publishing events?** If you publish then fail to persist, the event describes something that didn't happen. If you persist then fail to publish, the event is missing — but the transactional outbox pattern fixes that later. Persist-first is always safer.

**Why does `R` list both services?** It makes dependencies visible in the type signature. Any caller of `createBudget` that doesn't provide these services gets a compile error. No hidden globals, no `import` of a singleton.

### Adding an envelope

```typescript
// packages/budgeting/src/application/AddEnvelopeService.ts

export const addEnvelope = (
  budgetId: BudgetId,
  name: string,
  allocationCents: number,
): Effect.Effect<
  void,
  BudgetNotFound | DuplicateEnvelopeName | AllocationExceedsBudgetIncome | InvalidMoneyAmount,
  BudgetRepository | EventBus
> =>
  Effect.gen(function* () {
    const repo = yield* BudgetRepository
    const bus = yield* EventBus

    // Load aggregate — whole thing, not just part of it
    const budget = yield* repo.findById(budgetId)
    const allocation = yield* Money.make(allocationCents)

    // Domain method enforces invariants — if they fail, Effect short-circuits
    yield* budget.addEnvelope(name, allocation)

    // Save whole aggregate back
    yield* repo.save(budget)

    // Collect and publish events
    const events = budget.pullEvents()
    yield* Effect.forEach(events, bus.publish, { discard: true })
  })
```

Notice `addEnvelope` on `Budget` does the work. The application service only loads, delegates, saves, and publishes. If you find yourself writing `if` statements about business rules in an application service, move them to the domain.

---

## Step 5 — Transactions Bounded Context

**Layer: Domain (separate context)**

The Transactions context models recording of actual spending. It references the Budgeting context only through IDs — never through domain types. This is the **Anti-Corruption Layer** boundary.

```typescript
// packages/transactions/src/domain/Transaction.ts
import { Data, Effect, Option } from "effect"
import { TransactionId, EnvelopeRef } from "./ids"
import { Money, BudgetPeriod } from "shared-kernel"
import {
  InvalidTransactionAmount,
  TransactionAlreadyVoided,
  FutureDateNotAllowed,
} from "./errors"
import { TransactionRecorded, TransactionVoided, TransactionEvent } from "./events"

export type TransactionType = "expense" | "income"

export class Transaction {
  private _events: TransactionEvent[] = []
  private _voidedAt: Date | null = null

  constructor(
    readonly id: TransactionId,
    readonly envelopeRef: EnvelopeRef,  // just an ID string branded as EnvelopeRef
    readonly amount: Money,
    readonly type: TransactionType,
    readonly payee: string,
    readonly date: Date,
    readonly period: BudgetPeriod,
  ) {}

  get isVoided(): boolean { return this._voidedAt !== null }

  void(reason: string): Effect.Effect<void, TransactionAlreadyVoided> {
    return Effect.gen(() => {
      if (this.isVoided)
        return yield* Effect.fail(new TransactionAlreadyVoided({ id: this.id }))
      this._voidedAt = new Date()
      this._events.push(new TransactionVoided({
        transactionId: this.id,
        envelopeRef: this.envelopeRef,
        amount: this.amount,
        type: this.type,
        reason,
        occurredAt: new Date(),
      }))
    })
  }

  pullEvents(): TransactionEvent[] {
    const events = [...this._events]
    this._events = []
    return events
  }

  static record(params: {
    envelopeRef: EnvelopeRef
    amount: Money
    type: TransactionType
    payee: string
    date: Date
    period: BudgetPeriod
  }): Effect.Effect<Transaction, InvalidTransactionAmount | FutureDateNotAllowed> {
    return Effect.gen(function* () {
      if (params.amount.isZero())
        return yield* Effect.fail(new InvalidTransactionAmount({ amount: params.amount }))
      if (params.date > new Date())
        return yield* Effect.fail(new FutureDateNotAllowed({ date: params.date }))

      const id = TransactionId(crypto.randomUUID())
      const tx = new Transaction(id, params.envelopeRef, params.amount, params.type, params.payee, params.date, params.period)
      tx._events.push(new TransactionRecorded({
        transactionId: id,
        envelopeRef: params.envelopeRef,
        amount: params.amount,
        type: params.type,
        occurredAt: new Date(),
      }))
      return tx
    })
  }
}
```

**Key design point:** `EnvelopeRef` is just a branded string — not an `Envelope` from the Budgeting package. Transactions don't import Budgeting types. They only know an envelope exists by ID. This is what keeps contexts decoupled.

---

## Step 6 — Cross-Context Event Flow

**Layer: Application (both contexts) + Infrastructure (event bus)**

When a transaction is recorded, the Budgeting context needs to update the envelope balance. This happens via domain events, not direct calls.

```
Transactions context                    Budgeting context
─────────────────────                   ─────────────────
RecordTransactionService
  → Transaction.record()
  → repo.save(tx)
  → bus.publish(TransactionRecorded)    ←── EventBus (infrastructure)
                                              │
                                        HandleTransactionRecordedService
                                          → repo.findById(budget)  ← finds budget by envelopeRef
                                          → budget.applySpend(amount)
                                          → repo.save(budget)
```

```typescript
// packages/budgeting/src/application/HandleTransactionRecordedService.ts
import { Effect } from "effect"
import { BudgetRepository } from "../ports/BudgetRepository"
import { TransactionRecorded } from "transactions/events"  // integration event — shared type or ACL translation

export const handleTransactionRecorded = (
  event: TransactionRecorded,
): Effect.Effect<void, never, BudgetRepository> =>
  Effect.gen(function* () {
    const repo = yield* BudgetRepository

    // EnvelopeRef in the event = EnvelopeId in our model — same string, different brand
    // This is where ACL translation happens if names diverge
    const envelopeId = EnvelopeId(event.envelopeRef)

    // We need to find which budget owns this envelope
    // In a real system: maintain an envelope-to-budget index
    // For weekend: scan all budgets for the period (or store envelopeId → budgetId separately)
    const budget = yield* repo.findByEnvelopeId(envelopeId)
    if (budget === null) return  // envelope orphaned — log and ignore

    yield* budget.applySpend(envelopeId, event.amount)
    yield* repo.save(budget)
  })
```

**Layer interaction:** The application service in Budgeting reacts to an event published by Transactions. Neither context calls the other's services directly. The event bus is the only shared infrastructure.

---

## Step 7 — Event Bus infrastructure

**Layer: Infrastructure**

For the weekend, use Effect's built-in `PubSub` — no external broker needed.

```typescript
// packages/api/src/infra/InProcessEventBus.ts
import { Effect, Layer, PubSub } from "effect"
import { EventBus, DomainEvent } from "shared-kernel"
import { handleTransactionRecorded } from "budgeting/application"

export const InProcessEventBusLive = Layer.scoped(
  EventBus,
  Effect.gen(function* () {
    const hub = yield* PubSub.unbounded<DomainEvent>()

    // Subscriber: Budgeting reacts to Transactions events
    // This wiring happens here in infra — not in either context
    yield* Effect.forkScoped(
      Effect.gen(function* () {
        const sub = yield* PubSub.subscribe(hub)
        while (true) {
          const event = yield* Queue.take(sub)
          if (event._tag === "TransactionRecorded") {
            yield* handleTransactionRecorded(event).pipe(
              Effect.catchAll(() => Effect.void)  // don't crash the bus on handler errors
            )
          }
        }
      })
    )

    return EventBus.of({
      publish: (event) => PubSub.publish(hub, event).pipe(Effect.asVoid),
    })
  })
)
```

**Why wire subscriptions here (infra) and not in the contexts?** Because the connection between contexts is an infrastructure concern. Budgeting doesn't know Transactions exists. Transactions doesn't know Budgeting exists. Only the composition root — the API package — knows about both and wires them.

---

## Step 8 — API Layer

**Layer: Presentation**

The HTTP layer is deliberately thin. It validates HTTP input, calls an application service, and maps domain errors to HTTP status codes. No business logic.

```typescript
// packages/api/src/routes/budgets.ts
import { HttpRouter, HttpServerRequest, HttpServerResponse } from "@effect/platform"
import { Effect } from "effect"
import { createBudget } from "budgeting/application"
import { BudgetAlreadyExists } from "budgeting/domain"

export const budgetRoutes = HttpRouter.empty.pipe(
  HttpRouter.post("/budgets", Effect.gen(function* () {
    const body = yield* HttpServerRequest.schemaBodyJson(CreateBudgetSchema)

    const budget = yield* createBudget({
      year: body.year,
      month: body.month,
      incomeCents: body.incomeCents,
    }).pipe(
      Effect.catchTag("BudgetAlreadyExists", () =>
        HttpServerResponse.json({ error: "Budget for this period already exists" }, { status: 409 })
      ),
    )

    return yield* HttpServerResponse.json({ id: budget.id, period: budget.period.label() }, { status: 201 })
  })),
)
```

**Layer interaction:** The route calls an application service. The application service's `R` type lists its dependencies. The HTTP server's main program provides those dependencies via `Layer.provide`. The route itself has no idea what database is being used.

---

## Step 9 — Layer composition (the wiring)

**Layer: Infrastructure (composition root)**

Everything wires together in `main.ts`. This is the only place that knows about every layer.

```typescript
// packages/api/src/main.ts
import { Layer } from "effect"
import { InMemoryBudgetRepositoryLive } from "budgeting/infra"
import { InMemoryTransactionRepositoryLive } from "transactions/infra"
import { InProcessEventBusLive } from "./infra/InProcessEventBus"

const AppLayer = Layer.mergeAll(
  InMemoryBudgetRepositoryLive,
  InMemoryTransactionRepositoryLive,
  InProcessEventBusLive,
)

// Later, swap InMemoryBudgetRepositoryLive → SqliteBudgetRepositoryLive
// Nothing else changes. That is the DDD payoff.
```

**This is where you see the full benefit.** Swapping persistence is one line. Every layer above it — application services, domain logic, HTTP routes — is unchanged because they depend only on interfaces, never on implementations.

---

## How layers call each other — complete trace

Concrete example: user submits a transaction for "Groceries" envelope.

```
POST /transactions  { envelopeRef, amount: 4200, payee: "Lidl", date: "2026-08-10" }
      │
      ▼ [Presentation] Parse HTTP body, call application service
      │
RecordTransactionService
  yield* TransactionRepository        ← asks Effect for the service (injected via Layer)
  yield* Money.make(4200)             ← [Domain] value object validation
  yield* Transaction.record(params)   ← [Domain] aggregate factory, invariant checks
  yield* repo.save(tx)                ← [Infrastructure] in-memory store
  yield* bus.publish(TransactionRecorded) ← [Infrastructure] PubSub
      │
      │ (async, in background fiber)
      ▼
InProcessEventBus subscriber wakes
  event._tag === "TransactionRecorded"
  yield* handleTransactionRecorded(event)
      │
      ▼ [Application, Budgeting context]
HandleTransactionRecordedService
  yield* BudgetRepository             ← asks Effect for its own repo
  yield* repo.findByEnvelopeId(id)    ← [Infrastructure] in-memory scan
  yield* budget.applySpend(id, amount) ← [Domain] aggregate method, updates balance
  yield* repo.save(budget)            ← [Infrastructure] write back
      │
      ▼
HTTP 201 response returns to user
Envelope balance updated in memory
```

No layer skips another. Domain never touches infrastructure. Infrastructure never contains business logic. Application services never talk to each other directly — only through events or via the composition root.

---

## What to build first (and why that order)

1. **Shared kernel** — unblocks everything else, zero dependencies
2. **Budgeting domain** — richest domain, most invariants, most learning per line of code
3. **Tests for domain** — they run with `Effect.runSync`, no setup needed, instant feedback
4. **Budgeting repos + application services** — now you can run the full flow in a test
5. **Transactions domain + tests** — same pattern, goes faster now
6. **API wiring** — connect HTTP to application services
7. **Frontend** — any fetch calls to the API; seeing balances update is the reward
8. **Swap to SQLite** — replace one `Layer`, verify tests still pass — this is the moment the architecture justifies itself

---

## Related

- [[domain-driven-design]]
- [[ddd-aggregate-root]]
- [[ddd-bounded-context]]
- [[ddd-domain-events]]
- [[ddd-repository-pattern]]
- [[ports-and-adapters]]
- [[effect-context-and-layers]]
