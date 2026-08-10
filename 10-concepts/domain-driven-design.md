---
title: Domain-Driven Design
type: concept
created: 2026-08-05
tags: [ddd, architecture, design, effect]
effect-source: 80-resources/effect-source
---

# Domain-Driven Design

## What

Approach to software development that centers the codebase on a **rich model of the business domain**. The model is the primary artifact — code reflects it, not the other way around.

Coined by Eric Evans in *Domain-Driven Design* (2003). Two distinct halves:

- **Strategic design** — how to carve up large systems into coherent pieces
- **Tactical design** — building blocks for implementing a single domain model

---

## Strategic Design

### Ubiquitous Language

Shared vocabulary between developers and domain experts. Every name in the code matches the language the business uses. No translation layer in your head.

> If the domain expert says "order is fulfilled", your code says `order.fulfill()`, not `order.setStatus("FULFILLED")`.

### Bounded Context

A boundary within which a model is internally consistent and the ubiquitous language is unambiguous.

> `Customer` in billing (has a payment method) vs `Customer` in shipping (has a delivery address) — two different models, both valid within their context.

→ See [[ddd-bounded-context]]

### Context Map

Diagram of all bounded contexts and their relationships — upstream/downstream, shared kernel, anti-corruption layer, etc.

---

## Tactical Design (Building Blocks)

### Entity

Defined by **identity**, not attributes. Two entities with the same data are still distinct objects. In Effect, use `Brand` for typed IDs:

```typescript
import { Brand, Data } from "effect"

type UserId = string & Brand.Brand<"UserId">
const UserId = Brand.nominal<UserId>()

class User extends Data.Class<{
  readonly id: UserId
  readonly email: Email
  readonly name: string
}> {}
```

### Value Object

Defined by **value**, not identity. Immutable. Two with the same data are interchangeable. Use `Brand` for primitives, `Data.Class` for composite values:

```typescript
import { Brand, Data } from "effect"

// Branded primitive — zero runtime cost
type Email = string & Brand.Brand<"Email">
const Email = Brand.nominal<Email>()

// Composite value object — structural equality via Data.Class
class Money extends Data.Class<{
  readonly amount: number
  readonly currency: "USD" | "EUR" | "GBP"
}> {}
```

`Data.Class` gives structural equality (`Equal.equals`) for free — two `Money` instances with the same fields are equal.

### Aggregate

Cluster of domain objects treated as a single unit. Has one **aggregate root** — the only entry point from outside. Transactions don't cross aggregate boundaries.

→ See [[ddd-aggregate-root]]

### Domain Event

Something that happened in the domain. Named in past tense. Use `Data.TaggedClass` so events are structurally equal and carry a discriminant `_tag`:

```typescript
import { Data } from "effect"

class OrderFulfilled extends Data.TaggedClass("OrderFulfilled")<{
  readonly orderId: OrderId
  readonly customerId: CustomerId
  readonly fulfilledAt: Date
}> {}
```

→ See [[ddd-domain-events]]

### Domain Errors

Use `Data.TaggedError` — gives structural equality, a `_tag` for `Effect.catchTag`, and makes errors yieldable:

```typescript
import { Data } from "effect"

class UserNotFound extends Data.TaggedError("UserNotFound")<{
  readonly id: UserId
}> {}

class OrderAlreadyFulfilled extends Data.TaggedError("OrderAlreadyFulfilled")<{
  readonly orderId: OrderId
}> {}
```

### Repository

Abstraction over persistence. Interface in the domain layer via `Context.Service`, implementation in infrastructure via `Layer`.

→ See [[ddd-repository-pattern]]

### Domain Service

Stateless operation that doesn't naturally belong on an entity. A plain Effect function with required services declared in its type:

```typescript
import { Effect } from "effect"
import { AccountRepository } from "./AccountRepository"

// The `AccountRepository` requirement is explicit in the return type
export const transferFunds = (
  fromId: AccountId,
  toId: AccountId,
  amount: Money
): Effect.Effect<void, InsufficientFunds | PersistenceError, AccountRepository> =>
  Effect.gen(function* () {
    const repo = yield* AccountRepository
    const from = yield* repo.findById(fromId)
    const to = yield* repo.findById(toId)
    yield* from.debit(amount)
    yield* to.credit(amount)
    yield* repo.save(from)
    yield* repo.save(to)
  })
```

### Application Service

Orchestrates use cases. Loads aggregates, calls domain logic, persists, publishes events. No business logic — only coordination. In Effect, this is a function returning an `Effect` with all infrastructure requirements in `R`:

```typescript
import { Effect } from "effect"
import { OrderRepository } from "../domain/order/OrderRepository"
import { EventBus } from "../infrastructure/EventBus"

export const fulfillOrder = (orderId: OrderId) =>
  Effect.gen(function* () {
    const repo = yield* OrderRepository
    const bus = yield* EventBus
    const order = yield* repo.findById(orderId)
    const events = order.fulfill()           // domain method returns events
    yield* repo.save(order)
    yield* Effect.forEach(events, bus.publish, { discard: true })
  })
```

---

## Layered Architecture

```
┌─────────────────────────────┐
│      Presentation / API     │
├─────────────────────────────┤
│     Application Services    │  ← orchestration, no business logic
├─────────────────────────────┤
│        Domain Layer         │  ← entities, value objects, aggregates,
│                             │    domain services, repository interfaces
├─────────────────────────────┤
│    Infrastructure Layer     │  ← DB, HTTP clients, repository impls (Layers)
└─────────────────────────────┘
```

In Effect: domain layer has **zero Effect requirements** in `R` (pure domain logic). Infrastructure provides `Layer` implementations. Application wires them together at the boundary.

→ See [[ports-and-adapters]]

---

## When DDD is worth it

Worth it when:
- The domain has **complex, non-trivial business logic**
- You have (or can get) domain expert access
- The system will live long enough to recoup the modeling investment

Overkill for:
- CRUD apps with no real business logic
- Simple data pipelines
- Short-lived scripts

---

## Key mental shift

Most developers model the **database**. DDD says model the **domain** — persistence is an implementation detail, not the center of the design.

---

## Related

- [[ddd-repository-pattern]]
- [[ddd-aggregate-root]]
- [[ddd-bounded-context]]
- [[ddd-domain-events]]
- [[ports-and-adapters]]
- [[unit-of-work-pattern]]
- [[effect-context-and-layers]]
- [[full-stack-layers]] — how DDD layers map to a complete deployed application

## References

- Evans, *Domain-Driven Design* (2003) — the original
- Vernon, *Implementing Domain-Driven Design* (2013) — more practical, focuses on strategic design
- Fowler, [DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- Effect source: `80-resources/effect-source/packages/effect/src/Brand.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Data.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Context.ts`
