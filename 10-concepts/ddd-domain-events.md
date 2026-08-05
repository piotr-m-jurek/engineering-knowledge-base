---
title: DDD Domain Events
type: concept
created: 2026-08-05
tags: [ddd, events, messaging, effect]
effect-source: 80-resources/effect-source
---

# DDD Domain Events

## What

A record of something that happened in the domain. Named in **past tense**. Immutable. Primary mechanism for cross-aggregate consistency and cross-context integration without direct coupling.

Use `Data.TaggedClass` — structural equality and a `_tag` discriminant for exhaustive matching:

```typescript
import { Data, Brand } from "effect"

type OrderId = string & Brand.Brand<"OrderId">
type CustomerId = string & Brand.Brand<"CustomerId">

class OrderFulfilled extends Data.TaggedClass("OrderFulfilled")<{
  readonly orderId: OrderId
  readonly customerId: CustomerId
  readonly occurredAt: Date
}> {}

class OrderCancelled extends Data.TaggedClass("OrderCancelled")<{
  readonly orderId: OrderId
  readonly reason: string
  readonly occurredAt: Date
}> {}

type OrderEvent = OrderFulfilled | OrderCancelled
```

## Why

Aggregates can't call each other's methods — that crosses aggregate boundaries and creates coupling. Instead, an aggregate emits an event; other parts of the system react.

> `Order.fulfill()` emits `OrderFulfilled` → `InventoryService` decrements stock → `NotificationService` emails the customer.

## Collect-and-dispatch pattern

Aggregate collects events internally; application service dispatches after persisting:

```typescript
import { Effect, Context, Layer } from "effect"

// --- Event Bus port (domain layer) ---

export interface EventBus {
  readonly publish: (event: DomainEvent) => Effect.Effect<void>
}

export class EventBus extends Context.Service<EventBus>()("EventBus") {}

// --- Application service wires it together ---

export const fulfillOrder = (orderId: OrderId) =>
  Effect.gen(function* () {
    const repo = yield* OrderRepository
    const bus = yield* EventBus

    const order = yield* repo.findById(orderId)
    yield* order.fulfill()                            // domain method, no I/O
    yield* repo.save(order)                           // persist first
    const events = order.pullEvents()                 // then collect events
    yield* Effect.forEach(events, bus.publish, { discard: true })
  })
```

## Transactional Outbox

For guaranteed at-least-once delivery without distributed transactions: save events to an `outbox` table **in the same DB transaction** as the aggregate, then publish asynchronously via a background worker.

```typescript
import { Effect, Layer } from "effect"

// Outbox-backed event bus implementation
export const OutboxEventBusLive = Layer.succeed(
  EventBus,
  EventBus.of({
    publish: (event) =>
      Effect.tryPromise({
        try: () => db.insert(outbox).values({
          id: crypto.randomUUID(),
          type: event._tag,
          payload: JSON.stringify(event),
          createdAt: new Date(),
          publishedAt: null,
        }),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(Effect.asVoid),
  })
)
```

A separate worker polls `outbox` for unpublished events and sends them to the message broker.

## Handling events with pattern matching

`Data.TaggedClass` events work with `Match` for exhaustive handling:

```typescript
import { Match } from "effect"

const handleOrderEvent = (event: OrderEvent) =>
  Match.value(event).pipe(
    Match.tag("OrderFulfilled", (e) => notifyCustomer(e.customerId)),
    Match.tag("OrderCancelled", (e) => restockInventory(e.orderId)),
    Match.exhaustive
  )
```

## Cross-context integration events

Between bounded contexts, domain events become **integration events** — published over a message bus (Kafka, etc). The receiving context translates them through its anti-corruption layer.

```typescript
// Billing context receives shipping's event, maps to its own model
const handleShippingEvent = (raw: ShippingContextEvent) =>
  Match.value(raw).pipe(
    Match.tag("ShipmentDelivered", (e) =>
      // translate to billing's own concept
      triggerInvoiceGeneration(CustomerId(e.recipientId))
    ),
    Match.exhaustive
  )
```

## Related

- [[domain-driven-design]]
- [[ddd-aggregate-root]] — aggregates emit events via `pullEvents()`
- [[ddd-repository-pattern]] — application service dispatches events after `repo.save()`
- [[ddd-bounded-context]] — integration events cross context boundaries
- [[unit-of-work-pattern]]

## References

- Evans, *DDD* ch. 8
- Vernon, *IDDD* ch. 8
- Effect source: `80-resources/effect-source/packages/effect/src/Data.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Context.ts`
