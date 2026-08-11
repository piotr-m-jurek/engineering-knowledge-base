---
title: DDD Finance App — Build Walkthrough
type: project
created: 2026-08-10
tags: [ddd, effect, finance, project, walkthrough]
effect-version: v4
effect-source: 80-resources/effect-source
---
x
# DDD Finance App — Build Walkthrough

Envelope budgeting app built to learn DDD structure. Rich domain, no infra complexity.

> Concepts: [[domain-driven-design]] · [[ddd-aggregate-root]] · [[ddd-bounded-context]] · [[ddd-domain-events]] · [[ddd-repository-pattern]] · [[ports-and-adapters]]

---

## Layer mental model — read this first

Every decision in the build maps to this diagram:

```
┌──────────────────────┐
│  Presentation / API  │  ← HTTP routes, parse input, map errors to status codes
└──────────┬───────────┘
           │ calls
           ▼
┌──────────────────────┐
│  Application Service │  ← orchestrate: load → domain → save → publish
└──────────┬───────────┘
           │ calls
           ▼
┌──────────────────────┐
│    Domain Layer      │  ← aggregates, value objects, pure, R = never
└──────────┬───────────┘
           │ interfaces only
           ▼
┌──────────────────────┐
│  Infrastructure      │  ← repo impls, event bus, DB, Layer.mergeAll
└──────────────────────┘
```

**One rule:** dependencies point downward only. Domain imports nothing from infrastructure. Infrastructure imports domain types but not vice versa. Application service is the only layer that touches both domain and infrastructure interfaces.

---

## Step 1 — Shared Kernel

**Layer: Domain (shared)**

Tiny, stable package. Contains only concepts with the exact same meaning in every bounded context. If you find yourself wanting to put something here that isn't obviously universal, it probably belongs in a specific context.

```
packages/shared-kernel/src/
  Money.ts
  BudgetPeriod.ts
  EntityId.ts
  DomainEvent.ts
  PersistenceError.ts
```

### Money — value object with smart constructor

```typescript
// packages/shared-kernel/src/Money.ts
import { Data, Effect } from "effect"

export class InvalidMoneyAmount extends Data.TaggedError("InvalidMoneyAmount")<{
  readonly cents: number
}> {}

export class Money extends Data.Class<{
  readonly cents: number    // integer, minor units — never floats
  readonly currency: "USD"
}> {
  // Smart constructor: domain rule enforced here, not in callers
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

**Why `Data.Class`?** Structural equality for free. `Equal.equals(new Money({cents:100, currency:"USD"}), new Money({cents:100, currency:"USD"}))` → `true`. Without it, two separate instances would never be equal (reference equality).

**Why integer cents?** `0.1 + 0.2 !== 0.3` in JS. Store in cents, display by dividing by 100.

**Why smart constructor returning `Effect`?** One place for the rule. Every caller goes through the same validation. Rule changes once.

### BudgetPeriod

```typescript
// packages/shared-kernel/src/BudgetPeriod.ts
import { Data, Effect } from "effect"

export class InvalidBudgetPeriod extends Data.TaggedError("InvalidBudgetPeriod")<{
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

  label(): string {
    return `${this.year}-${String(this.month).padStart(2, "0")}`
  }
}
```

### Branded IDs

```typescript
// packages/shared-kernel/src/EntityId.ts
import { Brand } from "effect"

// Factory: call once per entity type to get a branded ID constructor
export const makeIdBrand = <Tag extends string>() =>
  Brand.nominal<string & Brand.Brand<Tag>>()
```

Usage in each context:
```typescript
// budgeting/src/domain/ids.ts
import { Brand } from "effect"
import { makeIdBrand } from "shared-kernel"

export type BudgetId = string & Brand.Brand<"BudgetId">
export const BudgetId = makeIdBrand<"BudgetId">()

export type EnvelopeId = string & Brand.Brand<"EnvelopeId">
export const EnvelopeId = makeIdBrand<"EnvelopeId">()
```

**Why branded IDs?** `repo.findById(userId)` won't accidentally accept an `orderId`. TypeScript rejects the wrong brand at compile time — no runtime check needed.

### EventBus port (service defined in shared kernel)

```typescript
// packages/shared-kernel/src/EventBus.ts
import { Context, Effect } from "effect"

export interface DomainEvent {
  readonly _tag: string
  readonly occurredAt: Date
}

// v4: Context.Service replaces Context.Tag
export class EventBus extends Context.Service<EventBus, {
  readonly publish: (event: DomainEvent) => Effect.Effect<void>
}>()("shared-kernel/EventBus") {}
```

**Layer interaction:** Domain and application layers both import this interface. Infrastructure provides the implementation. The string key `"shared-kernel/EventBus"` is what Effect uses at runtime to resolve the service — pick a stable, namespaced string.

---

## Step 2 — Budgeting Domain Layer

**Layer: Domain**

Pure TypeScript. No database, no HTTP. Effect used only for typed error returns — `R` is always `never` on domain methods.

```
packages/budgeting/src/domain/
  ids.ts
  errors.ts
  events.ts
  Envelope.ts    ← child entity
  Budget.ts      ← aggregate root
```

### Domain errors

```typescript
// packages/budgeting/src/domain/errors.ts
import { Data } from "effect"
import { Money, BudgetPeriod } from "shared-kernel"
import { BudgetId, EnvelopeId } from "./ids"

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

**Why `Data.TaggedError`?**
1. `_tag` discriminant → `Effect.catchTag("AllocationExceedsBudgetIncome", ...)` — exhaustive matching
2. Structural equality — two errors with same fields are equal (useful in tests)
3. Yieldable — `return yield* new EnvelopeNotFound({ id })` inside `Effect.gen` without wrapping in `Effect.fail`

### Domain events

```typescript
// packages/budgeting/src/domain/events.ts
import { Data } from "effect"
import { Money, BudgetPeriod, DomainEvent } from "shared-kernel"
import { BudgetId, EnvelopeId } from "./ids"

export class BudgetCreated extends Data.TaggedClass("BudgetCreated")<{
  readonly budgetId: BudgetId
  readonly period: BudgetPeriod
  readonly income: Money
  readonly occurredAt: Date
}> implements DomainEvent {}

export class EnvelopeCreated extends Data.TaggedClass("EnvelopeCreated")<{
  readonly budgetId: BudgetId
  readonly envelopeId: EnvelopeId
  readonly name: string
  readonly allocation: Money
  readonly occurredAt: Date
}> implements DomainEvent {}

export class FundsAllocated extends Data.TaggedClass("FundsAllocated")<{
  readonly budgetId: BudgetId
  readonly envelopeId: EnvelopeId
  readonly newAllocation: Money
  readonly occurredAt: Date
}> implements DomainEvent {}

export type BudgetingEvent = BudgetCreated | EnvelopeCreated | FundsAllocated
```

**Why `Data.TaggedClass` not `Data.TaggedError`?** Events are data, not errors. They record what happened. `Data.TaggedClass` gives structural equality and a discriminant tag.

### Envelope entity (child)

```typescript
// packages/budgeting/src/domain/Envelope.ts
import { Data } from "effect"
import { Money } from "shared-kernel"
import { EnvelopeId } from "./ids"

// Internal to Budget aggregate — never held directly from outside
export class Envelope extends Data.Class<{
  readonly id: EnvelopeId
  readonly name: string
  readonly allocation: Money  // budgeted amount
  readonly balance: Money     // remaining after spend
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

`Envelope` is an entity — it has identity (`EnvelopeId`) and mutates over time. It is a **child entity**: external code holds `BudgetId`, not `EnvelopeId`. You reach envelopes only through `Budget`.

### Budget aggregate root

```typescript
// packages/budgeting/src/domain/Budget.ts
import { Effect, HashMap, Option } from "effect"
import { Money, BudgetPeriod } from "shared-kernel"
import { BudgetId, EnvelopeId } from "./ids"
import { Envelope } from "./Envelope"
import {
  AllocationExceedsBudgetIncome, DuplicateEnvelopeName, EnvelopeNotFound,
} from "./errors"
import { BudgetingEvent, EnvelopeCreated, FundsAllocated, BudgetCreated } from "./events"

export class Budget {
  private _events: BudgetingEvent[] = []

  constructor(
    readonly id: BudgetId,
    readonly period: BudgetPeriod,
    readonly income: Money,
    private _envelopes: HashMap.HashMap<EnvelopeId, Envelope>,
  ) {}

  get envelopes(): HashMap.HashMap<EnvelopeId, Envelope> {
    return this._envelopes  // HashMap is immutable — safe to expose directly
  }

  // --- Domain methods ---

  // Invariant: name unique within budget
  // Invariant: total allocation ≤ income after adding
  addEnvelope(
    name: string,
    allocation: Money,
  ): Effect.Effect<void, DuplicateEnvelopeName | AllocationExceedsBudgetIncome> {
    return Effect.gen(() => {
      const nameTaken = HashMap.some(this._envelopes, (e) => e.name === name)
      if (nameTaken)
        return yield* new DuplicateEnvelopeName({ name })  // v4: yieldable directly

      const newTotal = this.totalAllocated().add(allocation)
      if (newTotal.cents > this.income.cents)
        return yield* new AllocationExceedsBudgetIncome({
          requested: allocation,
          available: this.income.subtract(this.totalAllocated()),
        })

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
      const found = HashMap.get(this._envelopes, envelopeId)
      if (Option.isNone(found))
        return yield* new EnvelopeNotFound({ id: envelopeId })

      const otherTotal = this.totalAllocated().subtract(found.value.allocation)
      const newTotal = otherTotal.add(newAllocation)
      if (newTotal.cents > this.income.cents)
        return yield* new AllocationExceedsBudgetIncome({
          requested: newAllocation,
          available: this.income.subtract(otherTotal),
        })

      this._envelopes = HashMap.set(
        this._envelopes,
        envelopeId,
        found.value.withAllocation(newAllocation),
      )
      this._events.push(new FundsAllocated({
        budgetId: this.id,
        envelopeId,
        newAllocation,
        occurredAt: new Date(),
      }))
    })
  }

  // Called from Transactions context via event — not via direct import
  applySpend(envelopeId: EnvelopeId, amount: Money): Effect.Effect<void, EnvelopeNotFound> {
    return Effect.gen(() => {
      const found = HashMap.get(this._envelopes, envelopeId)
      if (Option.isNone(found))
        return yield* new EnvelopeNotFound({ id: envelopeId })
      const updated = found.value.withBalance(found.value.balance.subtract(amount))
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

  // Application service calls this AFTER repo.save() — never before
  pullEvents(): BudgetingEvent[] {
    const events = [...this._events]
    this._events = []
    return events
  }

  // Factory — only way to create a new Budget
  static create(period: BudgetPeriod, income: Money): Budget {
    const id = BudgetId(crypto.randomUUID())
    const budget = new Budget(id, period, income, HashMap.empty())
    budget._events.push(new BudgetCreated({ budgetId: id, period, income, occurredAt: new Date() }))
    return budget
  }
}
```

What this demonstrates:

- All invariants enforced in the aggregate root — no caller can bypass them
- `_envelopes` is private — the only entry point is `addEnvelope()` which checks rules
- Events collected internally, released via `pullEvents()` — timing is controlled by the application layer
- `R` is always `never` on every method — zero infrastructure in the domain
- `yield* new DomainError(...)` — v4 yieldable errors, no `Effect.fail()` wrapper needed

### Tests — no infrastructure required

```typescript
// packages/budgeting/src/domain/__tests__/Budget.test.ts
import { Effect } from "effect"
import { expect, test } from "vitest"
import { Budget } from "../Budget"
import { Money, BudgetPeriod } from "shared-kernel"

const period = BudgetPeriod.make(2026, 8)
const income = Money.make(300_000)  // $3,000.00

test("cannot allocate more than income", () =>
  Effect.gen(function* () {
    const budget = Budget.create(
      yield* period,
      yield* income,
    )
    yield* budget.addEnvelope("Rent", yield* Money.make(200_000))
    const result = yield* budget.addEnvelope("Everything Else", yield* Money.make(200_000))
      .pipe(Effect.either)  // capture Left/Right instead of failing the test

    expect(result._tag).toBe("Left")
    if (result._tag === "Left")
      expect(result.left._tag).toBe("AllocationExceedsBudgetIncome")
  }).pipe(Effect.runPromise)
)

test("duplicate envelope names rejected", () =>
  Effect.gen(function* () {
    const budget = Budget.create(yield* period, yield* income)
    yield* budget.addEnvelope("Groceries", yield* Money.make(50_000))
    const result = yield* budget.addEnvelope("Groceries", yield* Money.make(30_000))
      .pipe(Effect.either)

    expect(result._tag).toBe("Left")
  }).pipe(Effect.runPromise)
)
```

Pure domain tests run with `Effect.runPromise` — no setup, no mocks, instant feedback.

---

## Step 3 — Repository port

**Layer: Domain (interface) / Infrastructure (implementation)**

The domain defines what it needs from persistence as a pure interface. It never knows whether data is in SQLite, Postgres, or memory.

```typescript
// packages/budgeting/src/ports/BudgetRepository.ts
import { Context, Effect } from "effect"
import { Budget } from "../domain/Budget"
import { BudgetId } from "../domain/ids"
import { BudgetNotFound } from "../domain/errors"
import { BudgetPeriod, PersistenceError } from "shared-kernel"

export interface IBudgetRepository {
  findById(id: BudgetId): Effect.Effect<Budget, BudgetNotFound | PersistenceError>
  findByPeriod(period: BudgetPeriod): Effect.Effect<Budget | null, PersistenceError>
  findByEnvelopeId(envelopeId: EnvelopeId): Effect.Effect<Budget | null, PersistenceError>
  save(budget: Budget): Effect.Effect<void, PersistenceError>
}

// v4: Context.Service replaces Context.Tag
export class BudgetRepository extends Context.Service<BudgetRepository, IBudgetRepository>()(
  "budgeting/BudgetRepository"
) {}
```

**Layer interaction:** Domain defines the shape. Infrastructure provides the object. Application service uses `yield* BudgetRepository` to ask for it — Effect resolves it from the `Layer` at runtime. Domain never imports from infrastructure.

### In-memory implementation

```typescript
// packages/budgeting/src/infra/InMemoryBudgetRepository.ts
import { Effect, Layer, Ref, HashMap, Option, Equal } from "effect"
import { BudgetRepository } from "../ports/BudgetRepository"
import { Budget } from "../domain/Budget"
import { BudgetId, EnvelopeId } from "../domain/ids"
import { BudgetNotFound } from "../domain/errors"
import { BudgetPeriod } from "shared-kernel"

export const InMemoryBudgetRepositoryLayer = Layer.effect(
  BudgetRepository,
  Effect.gen(function* () {
    // Ref: Effect's mutable reference — safe across fibers
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
            const entry = HashMap.findFirst(map, (b) => Equal.equals(b.period, period))
            return Option.match(entry, { onNone: () => null, onSome: ([, b]) => b })
          })
        ),

      findByEnvelopeId: (envelopeId) =>
        Ref.get(store).pipe(
          Effect.map((map) => {
            const entry = HashMap.findFirst(map, (b) =>
              HashMap.has(b.envelopes, envelopeId)
            )
            return Option.match(entry, { onNone: () => null, onSome: ([, b]) => b })
          })
        ),

      save: (budget) =>
        Ref.update(store, HashMap.set(budget.id, budget)),
    })
  })
)
```

**Layer interaction:** `Layer.effect(BudgetRepository, ...)` says "to satisfy `BudgetRepository`, run this effect to produce the implementation." The `Ref` is created once at layer initialization and shared. `Layer.effect` in v4 handles `Scope` automatically — no `Layer.scoped` needed.

---

## Step 4 — Application Services

**Layer: Application**

Orchestration only. Load aggregate → call domain method → save → collect events → publish. No `if` about business rules — those live in the domain.

```typescript
// packages/budgeting/src/application/CreateBudgetService.ts
import { Effect } from "effect"
import { BudgetRepository } from "../ports/BudgetRepository"
import { Budget } from "../domain/Budget"
import { BudgetAlreadyExists } from "../domain/errors"
import { EventBus, Money, BudgetPeriod } from "shared-kernel"

// R lists dependencies explicitly — callers must provide them
export const createBudget = Effect.fn("createBudget")(function* (
  year: number,
  month: number,
  incomeCents: number,
): Effect.Effect<
  Budget,
  BudgetAlreadyExists | InvalidMoneyAmount | InvalidBudgetPeriod,
  BudgetRepository | EventBus
> {
  const repo = yield* BudgetRepository
  const bus = yield* EventBus

  const period = yield* BudgetPeriod.make(year, month)
  const income = yield* Money.make(incomeCents)

  const existing = yield* repo.findByPeriod(period)
  if (existing !== null)
    return yield* new BudgetAlreadyExists({ period })

  const budget = Budget.create(period, income)

  // Persist first — then publish events
  // If persist fails, no event is published. Safe.
  // If publish fails after persist, transactional outbox fixes this later.
  yield* repo.save(budget)
  const events = budget.pullEvents()
  yield* Effect.forEach(events, bus.publish, { discard: true })

  return budget
})
```

**Why `Effect.fn("createBudget")`?** v4 preferred pattern for named effectful functions. Adds automatic span (tracing) and preserves stack traces. The string label shows up in traces.

```typescript
// packages/budgeting/src/application/AddEnvelopeService.ts
import { Effect } from "effect"
import { BudgetId, EnvelopeId } from "../domain/ids"
import { BudgetRepository } from "../ports/BudgetRepository"
import { EventBus, Money } from "shared-kernel"

export const addEnvelope = Effect.fn("addEnvelope")(function* (
  budgetId: BudgetId,
  name: string,
  allocationCents: number,
): Effect.Effect<
  void,
  BudgetNotFound | DuplicateEnvelopeName | AllocationExceedsBudgetIncome | InvalidMoneyAmount,
  BudgetRepository | EventBus
> {
  const repo = yield* BudgetRepository
  const bus = yield* EventBus

  // Load whole aggregate — always whole, never partial
  const budget = yield* repo.findById(budgetId)
  const allocation = yield* Money.make(allocationCents)

  // Domain method enforces invariants
  // If they fail, Effect short-circuits — save never happens
  yield* budget.addEnvelope(name, allocation)

  yield* repo.save(budget)
  const events = budget.pullEvents()
  yield* Effect.forEach(events, bus.publish, { discard: true })
})
```

**The rule:** if you find yourself writing `if` statements about business rules in an application service, that logic belongs in the domain aggregate instead.

---

## Step 5 — Transactions Bounded Context

**Layer: Domain (separate context)**

Transactions context models recording of actual spending. It references the Budgeting context only through IDs — never through domain types. This is the Anti-Corruption Layer boundary in practice.

```typescript
// packages/transactions/src/domain/ids.ts
import { Brand } from "effect"
import { makeIdBrand } from "shared-kernel"

export type TransactionId = string & Brand.Brand<"TransactionId">
export const TransactionId = makeIdBrand<"TransactionId">()

// EnvelopeRef: an ID pointing to an envelope in Budgeting context
// NOT the Budgeting EnvelopeId type — a separate brand
// Same string value, different type — keeps contexts decoupled
export type EnvelopeRef = string & Brand.Brand<"EnvelopeRef">
export const EnvelopeRef = makeIdBrand<"EnvelopeRef">()
```

```typescript
// packages/transactions/src/domain/Transaction.ts
import { Effect } from "effect"
import { Money, BudgetPeriod, DomainEvent } from "shared-kernel"
import { TransactionId, EnvelopeRef } from "./ids"
import {
  InvalidTransactionAmount, TransactionAlreadyVoided, FutureDateNotAllowed,
} from "./errors"
import { TransactionRecorded, TransactionVoided, TransactionEvent } from "./events"

export type TransactionType = "expense" | "income"

export class Transaction {
  private _events: TransactionEvent[] = []
  private _voidedAt: Date | null = null

  constructor(
    readonly id: TransactionId,
    readonly envelopeRef: EnvelopeRef,
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
        return yield* new TransactionAlreadyVoided({ id: this.id })
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

  // Factory — only way to record a transaction
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
        return yield* new InvalidTransactionAmount({ amount: params.amount })
      if (params.date > new Date())
        return yield* new FutureDateNotAllowed({ date: params.date })

      const id = TransactionId(crypto.randomUUID())
      const tx = new Transaction(
        id, params.envelopeRef, params.amount,
        params.type, params.payee, params.date, params.period,
      )
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

**Key design point:** `EnvelopeRef` is a separate branded type from Budgeting's `EnvelopeId` — same underlying string, different brand. Transactions never import from the Budgeting package. This is what keeps contexts decoupled: they share IDs by convention, not by type import.

---

## Step 6 — Cross-context event flow

**Layer: Application (both contexts) + Infrastructure (event bus)**

When a transaction is recorded, Budgeting needs to update the envelope balance. Done via events, not direct calls.

```
Transactions context                    Budgeting context
─────────────────────                   ─────────────────
RecordTransactionService
  → Transaction.record()
  → repo.save(tx)
  → bus.publish(TransactionRecorded)    ──▶  EventBus
                                               │
                                        HandleTransactionRecorded
                                          → repo.findByEnvelopeId(...)
                                          → budget.applySpend(amount)
                                          → repo.save(budget)
```

```typescript
// packages/budgeting/src/application/HandleTransactionRecorded.ts
import { Effect } from "effect"
import { BudgetRepository } from "../ports/BudgetRepository"
import { EnvelopeId } from "../domain/ids"

// Import from shared-kernel or a shared integration events package — not from transactions domain
import type { TransactionRecorded } from "shared-kernel/integration-events"

export const handleTransactionRecorded = Effect.fn("handleTransactionRecorded")(function* (
  event: TransactionRecorded,
): Effect.Effect<void, never, BudgetRepository> {
  const repo = yield* BudgetRepository

  // ACL translation: EnvelopeRef → EnvelopeId
  // Same string value, rebranded to local context's type
  const envelopeId = EnvelopeId(event.envelopeRef)

  const budget = yield* repo.findByEnvelopeId(envelopeId)
  if (budget === null) return  // envelope orphaned — safe to ignore

  yield* budget.applySpend(envelopeId, event.amount)
  yield* repo.save(budget)
})
```

**Layer interaction:** Budgeting's application layer reacts to an event published by Transactions. Neither context calls the other's services directly. The event bus (infrastructure) is the only connection. The translation point (`EnvelopeRef` → `EnvelopeId`) is the Anti-Corruption Layer in code.

---

## Step 7 — Event Bus infrastructure

**Layer: Infrastructure**

In-process bus using Effect's `PubSub`. No external broker for the weekend.

```typescript
// packages/api/src/infra/InProcessEventBus.ts
import { Effect, Layer, PubSub, Stream } from "effect"
import { EventBus, DomainEvent } from "shared-kernel"
import { handleTransactionRecorded } from "budgeting/application"

export const InProcessEventBusLayer = Layer.effect(
  EventBus,
  Effect.gen(function* () {
    // bounded: apply backpressure if consumers are slow
    const hub = yield* PubSub.bounded<DomainEvent>({ capacity: 256 })

    // Subscribe and route events to Budgeting context handler
    // forkScoped: fiber lives for the lifetime of this layer's scope
    yield* Effect.forkScoped(
      Stream.fromPubSub(hub).pipe(
        Stream.runForEach((event) => {
          if (event._tag === "TransactionRecorded") {
            return handleTransactionRecorded(event).pipe(
              Effect.catchAll(() => Effect.void)  // don't crash bus on handler errors
            )
          }
          return Effect.void
        })
      )
    )

    return EventBus.of({
      publish: (event) => PubSub.publish(hub, event).pipe(Effect.asVoid),
    })
  })
)
```

**Why wire subscriptions here (infra) and not in the contexts?** Because the connection between contexts is an infrastructure concern. Budgeting doesn't know Transactions exists. Transactions doesn't know Budgeting exists. Only the composition root knows about both and wires them. This is the **Separated Ways** relationship made explicit — neither context depends on the other in code.

**v4 pattern:** `Stream.fromPubSub` is the preferred way to consume a PubSub. Avoids raw `Queue.take` loops. `Effect.forkScoped` ties the fiber's lifetime to the layer's scope — it shuts down cleanly when the app exits.

---

## Step 8 — HTTP Layer

**Layer: Presentation**

Thin. Validates HTTP input, calls application service, maps domain errors to status codes. No business logic.

```typescript
// packages/api/src/routes/budgets.ts
import { HttpRouter, HttpServerRequest, HttpServerResponse } from "effect/unstable/http"
import { Effect, Schema } from "effect"
import { createBudget, addEnvelope } from "budgeting/application"
import { BudgetAlreadyExists, DuplicateEnvelopeName, AllocationExceedsBudgetIncome } from "budgeting/domain"

const CreateBudgetBody = Schema.Struct({
  year: Schema.Number,
  month: Schema.Number,
  incomeCents: Schema.Number,
})

const createBudgetHandler = Effect.gen(function* () {
  const req = yield* HttpServerRequest.HttpServerRequest
  const body = yield* req.json.pipe(
    Effect.flatMap(Schema.decodeUnknown(CreateBudgetBody))
  )

  const budget = yield* createBudget(body.year, body.month, body.incomeCents).pipe(
    Effect.catchTag("BudgetAlreadyExists", () =>
      HttpServerResponse.json(
        { error: "Budget for this period already exists" },
        { status: 409 }
      )
    ),
  )

  return HttpServerResponse.json(
    { id: budget.id, period: budget.period.label() },
    { status: 201 }
  )
})

// v4: route registration is layer-based
export const BudgetRoutes = HttpRouter.add("POST", "/budgets", createBudgetHandler)
```

**v4 HTTP:** `effect/unstable/http` replaces `@effect/platform`. Route registration returns a `Layer`, not a chained router value. This composes the same way as any other `Layer` — `Layer.mergeAll(BudgetRoutes, TransactionRoutes)`.

---

## Step 9 — Layer composition (the wiring)

**Layer: Infrastructure (composition root)**

Everything comes together in `main.ts`. The only place that knows about every layer.

```typescript
// packages/api/src/main.ts
import { Layer } from "effect"
import { HttpRouter } from "effect/unstable/http"
import { NodeHttpServer } from "@effect/platform-node"
import { InMemoryBudgetRepositoryLayer } from "budgeting/infra"
import { InMemoryTransactionRepositoryLayer } from "transactions/infra"
import { InProcessEventBusLayer } from "./infra/InProcessEventBus"
import { BudgetRoutes } from "./routes/budgets"
import { TransactionRoutes } from "./routes/transactions"

const RepositoriesLayer = Layer.mergeAll(
  InMemoryBudgetRepositoryLayer,
  InMemoryTransactionRepositoryLayer,
)

const AppLayer = Layer.mergeAll(
  RepositoriesLayer,
  InProcessEventBusLayer,
  BudgetRoutes,
  TransactionRoutes,
  NodeHttpServer.layer({ port: 3000 }),
)

// Later — swap repos, nothing else changes:
// InMemoryBudgetRepositoryLayer → SqliteBudgetRepositoryLayer
// That is the DDD payoff.
```

`Layer.mergeAll` composes layers. Effect resolves requirements at compile time — if a layer's dependency isn't provided, it's a type error, not a runtime crash.

---

## Complete request trace

Concrete example: POST `/transactions` — recording a £42 grocery spend.

```
POST /transactions  { envelopeRef, amount: 4200, payee: "Lidl", date: "2026-08-10", type: "expense" }
      │
      ▼  [Presentation] parse body, call application service
      │
RecordTransactionService
  yield* TransactionRepository              ← Effect resolves from Layer
  yield* Money.make(4200)                   ← [Domain] value object, InvalidMoneyAmount if non-integer
  yield* Transaction.record(params)         ← [Domain] aggregate factory:
                                                 - FutureDateNotAllowed if date > now
                                                 - InvalidTransactionAmount if zero
  yield* repo.save(tx)                      ← [Infrastructure] Ref.update in-memory
  yield* bus.publish(TransactionRecorded)   ← [Infrastructure] PubSub.publish
      │
      │  (concurrent fiber, same process)
      ▼
InProcessEventBus subscription wakes
  event._tag === "TransactionRecorded"
  yield* handleTransactionRecorded(event)
      │
      ▼  [Application, Budgeting context]
  const envelopeId = EnvelopeId(event.envelopeRef)   ← ACL translation
  yield* repo.findByEnvelopeId(envelopeId)            ← [Infrastructure] HashMap scan
  yield* budget.applySpend(envelopeId, amount)        ← [Domain] balance updated
  yield* repo.save(budget)                            ← [Infrastructure] write back
      │
      ▼
HTTP 201 returned to caller
Envelope balance reduced in memory
```

No layer skips another. Domain never touches infrastructure. Infrastructure never contains business logic. Application services never call each other directly — only through events.

---

## Implementation order

| Step | Why first |
|---|---|
| 1. Shared kernel | Unblocks everything — no deps, pure types |
| 2. Budgeting domain | Richest invariants, most learning per line |
| 3. Domain tests | `Effect.runPromise`, no setup — instant feedback loop |
| 4. Budgeting repos + app services | First full vertical slice working |
| 5. Transactions domain + tests | Same pattern, faster now |
| 6. API wiring | HTTP ↔ application services |
| 7. Frontend | Fetch calls to API — seeing balances update is the payoff |
| 8. SQLite swap | Replace one `Layer` — verify tests still pass. This is when the architecture justifies itself. |

---

## Related

- [[domain-driven-design]]
- [[ddd-aggregate-root]]
- [[ddd-bounded-context]]
- [[ddd-domain-events]]
- [[ddd-repository-pattern]]
- [[ports-and-adapters]]
- [[effect-context-and-layers]]

## References

- Effect v4 source: `80-resources/effect-source/packages/effect/src/`
- Effect v4 HTTP: `80-resources/effect-source/packages/effect/src/unstable/http/`
- Effect v4 LLMS doc: `80-resources/effect-source/LLMS.md`
- Effect v4 ai-docs: `80-resources/effect-source/ai-docs/`
