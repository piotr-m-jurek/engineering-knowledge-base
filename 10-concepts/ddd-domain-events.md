---
title: DDD Domain Events
type: concept
created: 2026-08-05
tags: [ddd, events, messaging]
---

# DDD Domain Events

## What

A record of something that happened in the domain. Named in **past tense**. Immutable. The primary mechanism for cross-aggregate consistency and cross-context integration without direct coupling.

```typescript
class OrderFulfilled {
  readonly occurredAt: Date
  constructor(readonly orderId: OrderId, readonly customerId: CustomerId) {
    this.occurredAt = new Date()
  }
}
```

## Why

Aggregates can't call each other's methods (that would cross aggregate boundaries and couple them). Instead, an aggregate emits an event; other aggregates/services react asynchronously.

> `Order.fulfill()` emits `OrderFulfilled` → `InventoryService` listens and decrements stock → `NotificationService` listens and emails the customer.

## Patterns

### Collect-and-dispatch
Aggregate collects events internally; application service dispatches after saving:

```typescript
class Order {
  private events: DomainEvent[] = []

  fulfill() {
    // ... invariant checks ...
    this.status = OrderStatus.Fulfilled
    this.events.push(new OrderFulfilled(this.id, this.customerId))
  }

  pullEvents(): DomainEvent[] {
    const e = [...this.events]
    this.events = []
    return e
  }
}
```

### Transactional outbox
Save events to an `outbox` table in the same transaction as the aggregate, then publish asynchronously. Guarantees at-least-once delivery without distributed transactions.

## Cross-context events

Between bounded contexts, domain events become **integration events** — published over a message bus (Kafka, RabbitMQ, etc). The receiving context translates them through its anti-corruption layer into its own model.

## Related

- [[domain-driven-design]]
- [[ddd-aggregate-root]] — aggregates emit events, never call other aggregates directly
- [[ddd-repository-pattern]] — application service dispatches events after repository save
- [[unit-of-work-pattern]]

## References

- Evans, *DDD* ch. 8
- Vernon, *IDDD* ch. 8
