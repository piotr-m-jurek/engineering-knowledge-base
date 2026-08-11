---
title: CQRS — Command Query Responsibility Segregation
type: concept
created: 2026-08-11
tags: [architecture, distributed-systems, cqrs, ddd, event-sourcing, effect]
---

# CQRS — Command Query Responsibility Segregation

> Separate the model you use to write state from the model you use to read it.

---

## Junior: "separate read and write methods"

You have a service with methods that both read and write. CQRS says: split them into two distinct objects.

- **Command** — mutates state, returns nothing (or just an ack). Has side effects.
- **Query** — reads state, returns data. Never mutates.

```typescript
// Before
class UserService {
  getUser(id: string): User { ... }
  updateEmail(id: string, email: string): void { ... }
}

// After CQRS — two separate services
class UserQueries {
  getUser(id: string): User { ... }
}
class UserCommands {
  updateEmail(id: string, email: string): void { ... }
}
```

Why? You know at a glance whether a method is safe to call repeatedly (query) or has side effects (command). The discipline prevents methods that read AND write — a common source of subtle bugs.

---

## Mid-level: "separate models for reads and writes"

It's not just method organization — reads and writes need **different data shapes**.

The **write side** deals with domain logic: validation, invariants, aggregates. It protects consistency.

The **read side** deals with presentation: denormalized, optimized for what queries need.

In Effect v4, you model this as two services with distinct contracts:

```typescript
import { Context, Effect, Layer } from "effect"
import { Schema } from "effect"

// --- domain types ---
class OrderId extends Schema.TaggedClass<OrderId>()("OrderId", {
  value: Schema.String,
}) {}

class AddItemError extends Schema.TaggedError<AddItemError>()(
  "AddItemError",
  { reason: Schema.String }
) {}

// Write-side view: the full aggregate, enforces invariants
interface OrderAggregate {
  readonly id: OrderId
  readonly items: ReadonlyArray<OrderItem>
  addItem(item: OrderItem): Effect.Effect<void, AddItemError>
}

// Read-side view: flat, fast, UI-shaped
interface OrderSummaryView {
  readonly orderId: string
  readonly customerName: string
  readonly itemCount: number
  readonly total: number
}

// --- commands service ---
class OrderCommands extends Context.Service<OrderCommands>()<
  "orders/OrderCommands",
  {
    addItem(orderId: OrderId, item: OrderItem): Effect.Effect<void, AddItemError>
    cancelOrder(orderId: OrderId): Effect.Effect<void, AddItemError>
  }
>()("orders/OrderCommands") {
  static readonly layer = Layer.effect(
    OrderCommands,
    Effect.gen(function* () {
      const repo = yield* OrderRepository
      return OrderCommands.of({
        addItem: Effect.fn("OrderCommands.addItem")(function* (orderId, item) {
          const order = yield* repo.findById(orderId)
          yield* order.addItem(item)
          yield* repo.save(order)
        }),
        cancelOrder: Effect.fn("OrderCommands.cancelOrder")(function* (orderId) {
          const order = yield* repo.findById(orderId)
          yield* order.cancel()
          yield* repo.save(order)
        }),
      })
    })
  )
}

// --- queries service ---
class OrderQueries extends Context.Service<OrderQueries>()<
  "orders/OrderQueries",
  {
    getSummary(orderId: OrderId): Effect.Effect<OrderSummaryView, OrderNotFound>
    listByCustomer(customerId: string): Effect.Effect<ReadonlyArray<OrderSummaryView>>
  }
>()("orders/OrderQueries") {
  static readonly layer = Layer.effect(
    OrderQueries,
    Effect.gen(function* () {
      const db = yield* ReadDatabase
      return OrderQueries.of({
        getSummary: Effect.fn("OrderQueries.getSummary")(function* (orderId) {
          return yield* db.queryOne(
            "SELECT * FROM order_summary_view WHERE order_id = ?",
            orderId.value
          )
        }),
        listByCustomer: Effect.fn("OrderQueries.listByCustomer")(function* (customerId) {
          return yield* db.query(
            "SELECT * FROM order_summary_view WHERE customer_id = ?",
            customerId
          )
        }),
      })
    })
  )
}
```

`OrderCommands` depends on the domain repository. `OrderQueries` depends on a read database — which could be a separate replica, a materialized view, or Redis. They don't share a dependency.

---

## Senior: "independent scaling + eventual consistency as a first-class concept"

With truly separate read/write paths, each side can evolve independently:

- **Scale reads** — read replicas, CDN caches, materialized views, Elasticsearch
- **Scale writes** — command queue, backpressure, saga coordination
- **Optimize each** — write side uses a relational DB for consistency; read side uses whatever is fastest to query

The mechanism that keeps them in sync is usually **domain events**:

```
POST /orders/:id/items
  → OrderCommands.addItem validates + persists write model
  → Emits ItemAddedToOrder event
  → Projector consumes event → updates order_summary table
  → GET /orders/:id/summary reads from summary (may lag briefly)
```

In Effect v4, the projector is just another service consuming an event stream:

```typescript
class OrderProjector extends Context.Service<OrderProjector>()<
  "orders/OrderProjector",
  {
    run(): Effect.Effect<never, never>
  }
>()("orders/OrderProjector") {
  static readonly layer = Layer.effect(
    OrderProjector,
    Effect.gen(function* () {
      const eventBus = yield* OrderEventBus
      const readDb = yield* ReadDatabase

      return OrderProjector.of({
        run: Effect.fn("OrderProjector.run")(function* () {
          yield* eventBus.subscribe.pipe(
            Effect.forEach(
              Effect.fn("OrderProjector.handleEvent")(function* (event) {
                switch (event._tag) {
                  case "ItemAddedToOrder":
                    yield* readDb.execute(
                      "UPDATE order_summary SET item_count = item_count + 1, total = ? WHERE order_id = ?",
                      event.newTotal,
                      event.orderId,
                    )
                    break
                  case "OrderCanceled":
                    yield* readDb.execute(
                      "UPDATE order_summary SET status = 'canceled' WHERE order_id = ?",
                      event.orderId,
                    )
                    break
                }
              })
            )
          )
        }),
      })
    })
  )
}
```

The trade-off is **eventual consistency**. After a command, the read model may lag. You must decide per use case: is a stale read acceptable here? Usually yes for dashboards. Never for payment confirmation.

CQRS at this level pairs naturally with [[event-sourcing]] (store events as source of truth, derive read models by projecting them) but they are **not the same thing** — you can do CQRS without event sourcing.

---

## Principal: "a boundary decision, not a pattern to apply everywhere"

CQRS introduces real costs:

| Cost | Details |
|---|---|
| Operational complexity | Two data paths to monitor, deploy, debug |
| Eventual consistency bugs | Cache invalidation, ordering guarantees, duplicate events |
| Developer cognitive load | "Which model do I touch? Which service do I inject?" |
| Infrastructure | Separate DBs, event bus, projectors, snapshot strategies |

**Where it earns its keep:**

- High read/write asymmetry (e.g. 1000:1 reads vs writes)
- Read models with fundamentally different shape than write models
- Audit/compliance requirements (ES + CQRS = full history by default)
- Write contention at scale — commands can be queued and processed serially without blocking reads

**Where it doesn't:**

- CRUD apps with no real domain logic — a single repository is enough
- Small teams where added complexity exceeds the benefit
- Greenfield with unvalidated load assumptions — YAGNI applies

The decision is a **[[ddd-bounded-context]] boundary call**. Apply CQRS inside a context where reads and writes have divergent needs. Don't apply it uniformly across the whole system because one subdomain needs it.

In Effect terms: the fact that `OrderCommands` and `OrderQueries` are separate services means their `Layer`s are independently replaceable. Swap `OrderQueries.layer` for a Redis-backed implementation with no change to commands — that's the architecture paying off. But if the app is a content management tool where every write is immediately followed by a read of the same data, you've just added indirection for no gain.

At the architectural level, CQRS is a **communication contract** between teams: the write team owns consistency, the read team owns projection efficiency. That separation of ownership is often the real reason to adopt it — not the performance wins.

---

## CQRS vs Event Sourcing

These are related but independent:

| | CQRS | Event Sourcing |
|---|---|---|
| Core idea | Separate read/write models | Persist events, derive state |
| Requires the other? | No | No |
| Common together? | Yes — ES produces events; CQRS projects them into read models |

You can have CQRS without ES (write model is a normal aggregate stored as current state; read model is a separate projected table). You can have ES without CQRS (store events, replay to answer all queries). Full ES + CQRS gives you both: immutable history and optimized read projections.

---

## Related

- [[event-sourcing]] — natural complement; ES produces the events CQRS projects
- [[ddd-domain-events]] — the events that flow from write side to read side
- [[ddd-aggregate-root]] — write model; aggregates enforce invariants, emit events
- [[ddd-repository-pattern]] — write side persists via repository
- [[ddd-bounded-context]] — CQRS is applied per-context, not globally
- [[ports-and-adapters]] — `OrderCommands`/`OrderQueries` are ports; `Layer` swaps adapters

## References

- Greg Young, *CQRS and Event Sourcing* (talk, 2010) — canonical introduction
- Martin Fowler, *CQRS* — https://martinfowler.com/bliki/CQRS.html
- Vernon, *IDDD* ch. 4 — CQRS within bounded contexts
- Udi Dahan, *Clarified CQRS* — https://udidahan.com/2009/12/09/clarified-cqrs/
