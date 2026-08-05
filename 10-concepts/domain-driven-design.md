---
title: Domain-Driven Design
type: concept
created: 2026-08-05
tags: [ddd, architecture, design]
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

A shared vocabulary between developers and domain experts. Every concept in the code — class names, method names, variable names — should match the language the business uses. No translation layer in your head.

> If the domain expert says "order is fulfilled", your code says `order.fulfill()`, not `order.setStatus("FULFILLED")`.

### Bounded Context

A boundary within which a model is internally consistent and the ubiquitous language is unambiguous. Large systems are split into multiple bounded contexts; the same word can mean different things in different contexts.

> `Customer` in a billing context (has a payment method) vs `Customer` in a shipping context (has a delivery address) — two different models, both valid within their context.

→ See [[ddd-bounded-context]]

### Context Map

A diagram of all bounded contexts and how they relate — upstream/downstream, shared kernel, anti-corruption layer, etc.

---

## Tactical Design (Building Blocks)

### Entity

An object defined by its **identity**, not its attributes. Two entities with the same data are still distinct objects.

```typescript
class User {
  constructor(readonly id: UserId, ...) {}
  // identity is id — changing email doesn't make it a different User
}
```

### Value Object

An object defined by its **value**, not identity. Immutable. Two value objects with the same data are interchangeable.

```typescript
class Email {
  constructor(readonly value: string) {}
  equals(other: Email) { return this.value === other.value }
}
```

No identity, no lifecycle. Model concepts like Money, Address, DateRange as value objects.

### Aggregate

A cluster of domain objects treated as a single unit for data changes. Has one **aggregate root** — the only entry point from outside. Transactions don't cross aggregate boundaries.

```
Order (aggregate root)
  └── OrderLine[]
  └── ShippingAddress (value object)
```

→ See [[ddd-aggregate-root]]

### Domain Event

Something that happened in the domain that other parts of the system might care about. Named in past tense. Used for cross-aggregate consistency without coupling.

```typescript
class OrderFulfilled {
  constructor(readonly orderId: OrderId, readonly at: Date) {}
}
```

→ See [[ddd-domain-events]]

### Repository

Abstraction over persistence. Interface in the domain layer, implementation in infrastructure. One repository per aggregate root.

→ See [[ddd-repository-pattern]]

### Domain Service

Stateless operation that doesn't naturally belong on an entity or value object. If a concept involves multiple aggregates or external systems, it's a domain service.

```typescript
// Not on User, not on Account — it's a cross-entity operation
class FundsTransferService {
  transfer(from: Account, to: Account, amount: Money): Effect.Effect<void, TransferError>
}
```

### Application Service

Orchestrates use cases. Loads aggregates via repositories, calls domain logic, persists results, publishes events. No business logic here — only coordination.

```typescript
class FulfillOrderService {
  execute(orderId: OrderId) {
    // 1. load aggregate
    // 2. call domain method
    // 3. save
    // 4. publish event
  }
}
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
│    Infrastructure Layer     │  ← DB, HTTP clients, repository impls
└─────────────────────────────┘
```

Dependency rule: **domain layer depends on nothing**. Infrastructure depends on domain (implements its interfaces). Application depends on domain.

→ See [[ports-and-adapters]] (Hexagonal Architecture — compatible approach)

---

## When DDD is worth it

DDD pays off when:
- The domain has **complex, non-trivial business logic**
- You have (or can get) access to domain experts
- The system will live long enough to recoup the modeling investment

DDD is overkill for:
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

## References

- Evans, *Domain-Driven Design* (2003) — the original
- Vernon, *Implementing Domain-Driven Design* (2013) — more practical, focuses on strategic design
- Fowler, [DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- Fowler, [BoundedContext](https://martinfowler.com/bliki/BoundedContext.html)
- Fowler, [DDD Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
