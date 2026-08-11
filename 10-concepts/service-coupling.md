---
title: Service Coupling and Decoupling
type: concept
created: 2026-08-11
tags: [architecture, coupling, event-driven, eda, effect, services, distributed-systems]
---

# Service Coupling and Decoupling

> "Services are coupled" → EDA (or just better API design)
> — [[architecture-overkill-guide]]

## What coupling means

Two services are **coupled** when one must know about the other's existence, interface, and availability to function. The most common form: service A calls service B directly.

Three dimensions:

| Dimension | What it means | Symptom |
|---|---|---|
| **Structural** | A imports / depends on B's type | Changing B's interface breaks A at compile time |
| **Temporal** | A and B must both be running right now | B down → A fails, even for unrelated operations |
| **Behavioral** | A's correctness depends on B's correctness | A bug in B causes A to fail or corrupt data |

All three usually come together when you do a direct service call.

---

## Level 1 — Junior: the direct call

In Effect, the most common form is one `Context.Service` `yield*`-ing another inside a `Layer.effect`:

```typescript
// From: 80-resources/effect-source/ai-docs/src/09_testing/20_layer-tests.ts
class TodoService extends Context.Service<TodoService, {
  addAndCount(title: string): Effect.Effect<number>
}>()("app/TodoService") {
  static readonly layerNoDeps = Layer.effect(
    TodoService,
    Effect.gen(function* () {
      const repo = yield* TodoRepo  // <-- direct dependency: structural + temporal coupling

      const addAndCount = Effect.fn("TodoService.addAndCount")(function* (title: string) {
        yield* repo.create(title)   // if TodoRepo is unavailable, this fails
        const todos = yield* repo.list
        return todos.length
      })

      return TodoService.of({ addAndCount })
    })
  )
}
```

`TodoService` can't exist without `TodoRepo`. They fail together, deploy together, test together.

This is **fine** when `TodoRepo` is an infrastructure adapter in the same deployment (a DB wrapper). The issue appears when both sides are **independent services** that should be able to fail or change independently.

---

## Level 2 — Mid: the problems it causes

**Temporal coupling in practice:**

```
User places order
  → OrderService calls InventoryService.reserve(itemId)
    → InventoryService is restarting (deploy in progress)
      → OrderService throws 503
        → User sees error, order never placed
```

Neither service had a bug. The coupling created a failure.

**Structural coupling in practice:**

```
InventoryService renames reserve(itemId) → hold(itemId, quantity: number)
  → must update OrderService at the same time
  → must deploy both at the same time
  → if one deploy fails, both are broken
```

**Behavioral coupling:** if `InventoryService` has a bug that returns wrong stock counts, `OrderService` makes wrong decisions based on those counts. One bug propagates across service boundaries.

---

## Level 3 — Senior: two escape routes

### Option A — Structural decoupling via ports (still sync)

Depend on an interface (a port), not a concrete service. This is [[ports-and-adapters]]:

```typescript
// port defined in OrderService's domain — owned by the caller
export class InventoryPort extends Context.Service<InventoryPort, {
  reserve(itemId: string, quantity: number): Effect.Effect<void, InventoryError>
}>()("orders/InventoryPort") {}

// OrderService depends on the port, not on InventoryService directly
class OrderService extends Context.Service<OrderService, {
  placeOrder(orderId: string, itemId: string): Effect.Effect<Order, OrderError>
}>()("orders/OrderService") {
  static readonly layer = Layer.effect(
    OrderService,
    Effect.gen(function* () {
      const inventory = yield* InventoryPort  // <-- interface, not concrete impl

      return OrderService.of({
        placeOrder: Effect.fn("OrderService.placeOrder")(function* (orderId, itemId) {
          yield* inventory.reserve(itemId, 1)
          // ...
        })
      })
    })
  )
}
```

- Removes structural coupling: `InventoryService` can change its internals freely
- Does **not** remove temporal coupling: `InventoryService` still must be up
- Great for testability: swap the port with a test double

### Option B — Temporal decoupling via events (async)

`OrderService` publishes an event and moves on. It doesn't call `InventoryService` at all.

Effect's `PubSub` is the in-process primitive for this pattern:

```typescript
// From: 80-resources/effect-source/ai-docs/src/01_effect/07_pubsub/10_pubsub.ts

export type OrderEvent =
  | { readonly _tag: "OrderPlaced"; readonly orderId: string }
  | { readonly _tag: "PaymentCaptured"; readonly orderId: string }
  | { readonly _tag: "OrderShipped"; readonly orderId: string }

export class OrderEvents extends Context.Service<OrderEvents, {
  publish(event: OrderEvent): Effect.Effect<void>
  readonly subscribe: Stream.Stream<OrderEvent>
}>()("acme/OrderEvents") {
  static readonly layer = Layer.effect(
    OrderEvents,
    Effect.gen(function* () {
      const pubsub = yield* PubSub.bounded<OrderEvent>({
        capacity: 256,
        replay: 50  // late subscribers catch up to last 50 events
      })

      yield* Effect.addFinalizer(() => PubSub.shutdown(pubsub))

      return OrderEvents.of({
        publish: Effect.fn("OrderEvents.publish")(function* (event) {
          yield* PubSub.publish(pubsub, event)
        }),
        subscribe: Stream.fromPubSub(pubsub)
      })
    })
  )
}
```

`OrderService` publishes `OrderPlaced`. `InventoryService` subscribes to `OrderEvents` and processes `OrderPlaced` when it gets to it. They never call each other.

**PubSub vs Queue distinction:**

| | `PubSub` | `Queue` |
|---|---|---|
| Delivery | Every subscriber gets every message (fan-out / broadcast) | One message → one consumer (work distribution) |
| Use for | Domain events with multiple handlers | Task processing, background jobs |

For cross-process EDA (services in separate deployments), `PubSub` maps to Kafka/RabbitMQ topics. For in-process, Effect's `PubSub.bounded` is the equivalent.

---

## Level 4 — Principal: which to pick

The question is not "sync or async?" but: **should these two things be allowed to fail together?**

```
Same team + same deploy cadence
  → structural decoupling (ports) is enough
  → EDA adds operational complexity for no independence gain

Different teams or different deploy schedules
  → temporal decoupling (EDA) buys real independence

Failure of B must not fail A
  → EDA: A publishes and moves on, B catches up when alive

Operation needs an immediate answer (user waiting for result)
  → stay sync, protect with timeout + circuit breaker

"We always deploy them together, they always change together"
  → they are not two services — merge them
```

### What EDA costs (the tradeoffs)

- **Eventual consistency**: `InventoryService` hasn't processed the event yet → stock count is stale
- **Idempotency**: at-least-once delivery means your consumer must handle duplicate events safely
- **Debugging**: a log in OrderService and a log in InventoryService don't automatically correlate — need distributed tracing
- **Schema evolution**: changing the event payload requires coordinating all consumers

Use the **Transactional Outbox** to guarantee at-least-once delivery without distributed transactions — see [[ddd-domain-events]] for the implementation with Effect.

---

## Decision matrix

| Symptom | Fix |
|---|---|
| Renaming a method in B breaks A | Port/interface abstraction |
| B down breaks A | EDA or circuit breaker + fallback |
| Testing A requires real B | Port abstraction + test double |
| B change forces A redeploy | EDA or contract testing |
| You deploy A and B always together | Merge them — they're one service |
| User needs an immediate answer | Sync call with retry + timeout |

---

## Related

- [[ports-and-adapters]] — structural decoupling via interfaces (hexagonal architecture)
- [[ddd-domain-events]] — domain events as the event payload; collect-and-dispatch pattern
- [[ddd-bounded-context]] — coupling between bounded contexts is where EDA most often appears
- [[architecture-overkill-guide]] — when NOT to use EDA
- [[event-sourcing]] — not the same as EDA but often confused with it
- [[durable-execution]] — when async reactions need guaranteed exactly-once processing
- [[process-manager-saga]] — coordinating multiple services without direct coupling

## Effect source references

- `80-resources/effect-source/ai-docs/src/01_effect/07_pubsub/10_pubsub.ts` — canonical `OrderEvents` PubSub service
- `80-resources/effect-source/ai-docs/src/01_effect/03_services/20_layer-composition.ts` — `Layer.provide` vs `Layer.provideMerge` for wiring dependencies
- `80-resources/effect-source/ai-docs/src/09_testing/20_layer-tests.ts` — testing coupled vs decoupled layers
- `80-resources/effect-source/packages/effect/src/PubSub.ts` — `bounded`, `dropping`, `sliding`, `unbounded` constructors
- `80-resources/effect-source/packages/effect/src/Queue.ts` — work-distribution alternative to PubSub
