---
title: Full-Stack Layers — Live Examples (Effect)
type: architecture
created: 2026-08-10
tags: [architecture, full-stack, effect, postgres, redis, react]
---

# Full-Stack Layers — Live Examples (Effect)

Concrete code for every layer in [[full-stack-layers]], using the same domain throughout: **an order system where users can place and fulfill orders**.

Stack: Effect + `@effect/platform` + Postgres (Drizzle) + Redis + React + `@effect/react`.

---

## Domain: the through-line

All examples operate on this aggregate and event — defined once, used everywhere.

```typescript
// domain/order/Order.ts
import { Brand, Data, Effect } from "effect"

export type OrderId = string & Brand.Brand<"OrderId">
export const OrderId = Brand.nominal<OrderId>()

export type CustomerId = string & Brand.Brand<"CustomerId">
export const CustomerId = Brand.nominal<CustomerId>()

export class OrderAlreadyFulfilled extends Data.TaggedError("OrderAlreadyFulfilled")<{
  readonly orderId: OrderId
}> {}

export class OrderFulfilled extends Data.TaggedClass("OrderFulfilled")<{
  readonly orderId: OrderId
  readonly customerId: CustomerId
  readonly occurredAt: Date
}> {}

export class Order {
  private _events: OrderFulfilled[] = []

  constructor(
    readonly id: OrderId,
    readonly customerId: CustomerId,
    private _status: "draft" | "fulfilled"
  ) {}

  get status() { return this._status }

  fulfill(): Effect.Effect<void, OrderAlreadyFulfilled> {
    return Effect.gen((_this => function* () {
      if (_this._status === "fulfilled")
        yield* Effect.fail(new OrderAlreadyFulfilled({ orderId: _this.id }))
      _this._status = "fulfilled"
      _this._events.push(new OrderFulfilled({
        orderId: _this.id,
        customerId: _this.customerId,
        occurredAt: new Date(),
      }))
    })())
  }

  pullEvents(): OrderFulfilled[] {
    const events = [...this._events]
    this._events = []
    return events
  }
}
```

---

## Layer 1 — Domain (pure logic)

The repository **interface** (port) lives in the domain. No imports from infrastructure.

```typescript
// domain/order/OrderRepository.ts
import { Context, Effect } from "effect"
import { Order, OrderId } from "./Order"
import { OrderNotFound, PersistenceError } from "./errors"

export interface OrderRepository {
  readonly findById: (id: OrderId) => Effect.Effect<Order, OrderNotFound | PersistenceError>
  readonly save: (order: Order) => Effect.Effect<void, PersistenceError>
}

export class OrderRepository extends Context.Service<OrderRepository>()("OrderRepository") {}
```

```typescript
// domain/order/EventBus.ts
import { Context, Effect } from "effect"
import { OrderFulfilled } from "./Order"

export interface EventBus {
  readonly publish: (event: OrderFulfilled) => Effect.Effect<void>
}

export class EventBus extends Context.Service<EventBus>()("EventBus") {}
```

Domain has **zero infrastructure imports**. These are just interfaces — shapes.

---

## Layer 2 — Application Service (orchestration)

Loads the aggregate, calls domain logic, saves, dispatches events. No SQL, no Redis, no HTTP.

```typescript
// application/FulfillOrderService.ts
import { Context, Effect, Layer } from "effect"
import { OrderId, OrderAlreadyFulfilled } from "../domain/order/Order"
import { OrderRepository } from "../domain/order/OrderRepository"
import { EventBus } from "../domain/order/EventBus"
import { OrderNotFound, PersistenceError } from "../domain/order/errors"

export interface FulfillOrderService {
  readonly execute: (
    orderId: OrderId
  ) => Effect.Effect<void, OrderNotFound | OrderAlreadyFulfilled | PersistenceError>
}

export class FulfillOrderService
  extends Context.Service<FulfillOrderService>()("FulfillOrderService") {}

export const FulfillOrderServiceLive = Layer.effect(
  FulfillOrderService,
  Effect.gen(function* () {
    const repo = yield* OrderRepository
    const bus = yield* EventBus

    return FulfillOrderService.of({
      execute: (orderId) =>
        Effect.gen(function* () {
          const order = yield* repo.findById(orderId)    // load
          yield* order.fulfill()                          // domain method
          yield* repo.save(order)                         // persist first
          const events = order.pullEvents()
          yield* Effect.forEach(events, bus.publish, { discard: true })
        }),
    })
  })
)
```

The type signature makes the requirements explicit: `R = OrderRepository | EventBus`. No hidden dependencies.

---

## Layer 3 — Backend HTTP (presentation boundary)

Decodes HTTP, lifts to domain types, calls application service, maps errors to HTTP status codes. No business logic.

```typescript
// api/OrdersApi.ts — contract declaration
import { HttpApi, HttpApiEndpoint, HttpApiGroup, HttpApiSchema } from "@effect/platform"
import { Schema } from "effect"
import { OrderAlreadyFulfilled } from "../domain/order/Order"
import { OrderNotFound } from "../domain/order/errors"

export class OrderNotFoundError extends Schema.TaggedError<OrderNotFoundError>()(
  "OrderNotFoundError",
  { id: Schema.String },
  HttpApiSchema.annotations({ status: 404 })
) {}

export class AlreadyFulfilledError extends Schema.TaggedError<AlreadyFulfilledError>()(
  "AlreadyFulfilledError",
  { orderId: Schema.String },
  HttpApiSchema.annotations({ status: 409 })
) {}

export class OrdersApi extends HttpApiGroup.make("orders")
  .add(
    HttpApiEndpoint.post("fulfill", "/:id/fulfill")
      .addSuccess(Schema.Void)
      .addError(OrderNotFoundError)
      .addError(AlreadyFulfilledError)
      .setPath(Schema.Struct({ id: Schema.String }))
  )
  .prefix("/orders") {}

export const Api = HttpApi.make("api").add(OrdersApi)
```

```typescript
// api/OrdersHandler.ts — handler implementation
import { HttpApiBuilder } from "@effect/platform"
import { Effect } from "effect"
import { Api, OrderNotFoundError, AlreadyFulfilledError } from "./OrdersApi"
import { FulfillOrderService } from "../application/FulfillOrderService"
import { OrderId } from "../domain/order/Order"

export const OrdersHandlerLive = HttpApiBuilder.group(Api, "orders", (handlers) =>
  handlers.handle("fulfill", ({ path }) =>
    Effect.gen(function* () {
      const service = yield* FulfillOrderService
      yield* service.execute(OrderId(path.id))
    }).pipe(
      Effect.catchTag("OrderNotFound", (e) =>
        Effect.fail(new OrderNotFoundError({ id: e.id }))
      ),
      Effect.catchTag("OrderAlreadyFulfilled", (e) =>
        Effect.fail(new AlreadyFulfilledError({ orderId: e.orderId }))
      )
    )
  )
)
```

The handler maps domain errors → API errors. That's its only logic.

---

## Layer 4 — Infrastructure: Database (Postgres + Drizzle)

The concrete `OrderRepository` implementation. Domain never imports this file.

```typescript
// infrastructure/persistence/schema.ts
import { pgTable, text, timestamp } from "drizzle-orm/pg-core"

export const orders = pgTable("orders", {
  id: text("id").primaryKey(),
  customerId: text("customer_id").notNull(),
  status: text("status", { enum: ["draft", "fulfilled"] }).notNull().default("draft"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
})
```

```typescript
// infrastructure/persistence/OrderMapper.ts
import { Order, OrderId, CustomerId } from "../../domain/order/Order"
import type { InferSelectModel } from "drizzle-orm"
import { orders } from "./schema"

type OrderRow = InferSelectModel<typeof orders>

export const toDomain = (row: OrderRow): Order =>
  new Order(OrderId(row.id), CustomerId(row.customerId), row.status)

export const toRecord = (order: Order): OrderRow => ({
  id: order.id,
  customerId: order.customerId,
  status: order.status,
  createdAt: new Date(),
})
```

```typescript
// infrastructure/persistence/DrizzleOrderRepository.ts
import { Effect, Layer } from "effect"
import { drizzle } from "drizzle-orm/node-postgres"
import { Pool } from "pg"
import { eq } from "drizzle-orm"
import { OrderRepository } from "../../domain/order/OrderRepository"
import { OrderNotFound, PersistenceError } from "../../domain/order/errors"
import { toDomain, toRecord } from "./OrderMapper"
import { orders } from "./schema"

const pool = new Pool({ connectionString: process.env.DATABASE_URL })
const db = drizzle(pool)

export const DrizzleOrderRepositoryLive = Layer.succeed(
  OrderRepository,
  OrderRepository.of({
    findById: (id) =>
      Effect.tryPromise({
        try: () => db.select().from(orders).where(eq(orders.id, id)),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(
        Effect.flatMap((rows) =>
          rows[0]
            ? Effect.succeed(toDomain(rows[0]))
            : Effect.fail(new OrderNotFound({ id }))
        )
      ),

    save: (order) =>
      Effect.tryPromise({
        try: () =>
          db.insert(orders)
            .values(toRecord(order))
            .onConflictDoUpdate({ target: orders.id, set: toRecord(order) }),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(Effect.asVoid),
  })
)
```

---

## Layer 5 — Infrastructure: Cache (Redis)

A `CachedOrderRepository` wraps `DrizzleOrderRepositoryLive` — same interface, adds a Redis read-through cache. The application service never knows.

```typescript
// infrastructure/cache/CachedOrderRepository.ts
import { Effect, Layer } from "effect"
import { createClient } from "redis"
import { OrderRepository } from "../../domain/order/OrderRepository"
import { Order, OrderId, CustomerId } from "../../domain/order/Order"
import { PersistenceError } from "../../domain/order/errors"

const redis = createClient({ url: process.env.REDIS_URL })
await redis.connect()

const TTL_SECONDS = 60

// Read-through: check Redis first, fall back to DB, populate cache
export const CachedOrderRepositoryLive = Layer.effect(
  OrderRepository,
  Effect.gen(function* () {
    const db = yield* OrderRepository   // inner: the Drizzle impl

    return OrderRepository.of({
      findById: (id) =>
        Effect.tryPromise({
          try: () => redis.get(`order:${id}`),
          catch: (e) => new PersistenceError({ cause: e }),
        }).pipe(
          Effect.flatMap((cached) => {
            if (cached) {
              const row = JSON.parse(cached)
              return Effect.succeed(new Order(OrderId(row.id), CustomerId(row.customerId), row.status))
            }
            return db.findById(id).pipe(
              Effect.tap((order) =>
                Effect.tryPromise({
                  try: () => redis.setEx(`order:${id}`, TTL_SECONDS, JSON.stringify({
                    id: order.id,
                    customerId: order.customerId,
                    status: order.status,
                  })),
                  catch: (e) => new PersistenceError({ cause: e }),
                })
              )
            )
          })
        ),

      // Write-through: save to DB, invalidate cache
      save: (order) =>
        db.save(order).pipe(
          Effect.tap(() =>
            Effect.tryPromise({
              try: () => redis.del(`order:${order.id}`),
              catch: (e) => new PersistenceError({ cause: e }),
            })
          )
        ),
    })
  })
)

// Wire: CachedOrderRepository reads from DrizzleOrderRepository internally
export const OrderRepositoryLive = CachedOrderRepositoryLive.pipe(
  Layer.provide(DrizzleOrderRepositoryLive)
)
```

Swap `DrizzleOrderRepositoryLive` for `OrderRepositoryLive` in `main.ts` — nothing else changes.

---

## Layer 6 — Infrastructure: Event Bus (Transactional Outbox)

Production-safe event delivery: write events to an `outbox` table **in the same transaction** as the aggregate. A background worker reads and publishes.

```typescript
// infrastructure/outbox/schema.ts
import { pgTable, text, timestamp, jsonb, boolean } from "drizzle-orm/pg-core"

export const outbox = pgTable("outbox", {
  id: text("id").primaryKey(),
  type: text("type").notNull(),             // event _tag
  payload: jsonb("payload").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  publishedAt: timestamp("published_at"),   // null = pending
})
```

```typescript
// infrastructure/outbox/OutboxEventBus.ts
import { Effect, Layer } from "effect"
import { EventBus } from "../../domain/order/EventBus"
import { PersistenceError } from "../../domain/order/errors"
import { db } from "../persistence/db"
import { outbox } from "./schema"

// Publish = insert into outbox table (same DB connection = same transaction)
export const OutboxEventBusLive = Layer.succeed(
  EventBus,
  EventBus.of({
    publish: (event) =>
      Effect.tryPromise({
        try: () =>
          db.insert(outbox).values({
            id: crypto.randomUUID(),
            type: event._tag,
            payload: event,
            createdAt: new Date(),
            publishedAt: null,
          }),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(Effect.asVoid),
  })
)
```

```typescript
// infrastructure/outbox/OutboxWorker.ts — background poller
import { Effect, Schedule, Layer } from "effect"
import { isNull } from "drizzle-orm"
import { db } from "../persistence/db"
import { outbox } from "./schema"

// In production: replace with Kafka/SQS publish inside the forEach
const publishToExternalBroker = (row: typeof outbox.$inferSelect) =>
  Effect.tryPromise({
    try: async () => {
      console.log(`[outbox] publishing ${row.type}`, row.payload)
      // e.g., await kafkaProducer.send({ topic: row.type, messages: [{ value: JSON.stringify(row.payload) }] })
      await db.update(outbox)
        .set({ publishedAt: new Date() })
        .where(/* eq(outbox.id, row.id) */ outbox.id.eq(row.id))
    },
    catch: (e) => new Error(String(e)),
  })

export const OutboxWorkerLive = Layer.scopedDiscard(
  Effect.gen(function* () {
    const poll = Effect.tryPromise({
      try: () => db.select().from(outbox).where(isNull(outbox.publishedAt)).limit(50),
      catch: (e) => new Error(String(e)),
    }).pipe(
      Effect.flatMap((rows) =>
        Effect.forEach(rows, publishToExternalBroker, { concurrency: 5, discard: true })
      )
    )

    yield* Effect.repeat(poll, Schedule.spaced("5 seconds")).pipe(
      Effect.forkScoped
    )
  })
)
```

Guarantees: if the app crashes after `repo.save()` but before publishing, the outbox row survives and the worker retries.

---

## Layer 7 — Wiring (main.ts)

All layers composed at the boundary. Business logic never sees `Layer.provide`.

```typescript
// main.ts
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { HttpApiBuilder, HttpRouter } from "@effect/platform"
import { Layer } from "effect"
import { Api } from "./api/OrdersApi"
import { OrdersHandlerLive } from "./api/OrdersHandler"
import { FulfillOrderServiceLive } from "./application/FulfillOrderService"
import { OrderRepositoryLive } from "./infrastructure/cache/CachedOrderRepository"
import { OutboxEventBusLive } from "./infrastructure/outbox/OutboxEventBus"
import { OutboxWorkerLive } from "./infrastructure/outbox/OutboxWorker"

const HttpLive = HttpRouter.serve(
  HttpApiBuilder.router(Api, [OrdersHandlerLive])
).pipe(
  Layer.provide(OrdersHandlerLive),
  Layer.provide(FulfillOrderServiceLive),
  Layer.provide(OrderRepositoryLive),   // Cache wrapping Drizzle
  Layer.provide(OutboxEventBusLive),
  Layer.provideMerge(NodeHttpServer.layer({ port: 3000 }))
)

const AppLive = Layer.mergeAll(HttpLive, OutboxWorkerLive)

NodeRuntime.runMain(Layer.launch(AppLive))
```

---

## Layer 8 — Frontend (React + Effect)

`@effect/react` + `useQuery`-style hook that runs an `Effect` and exposes loading/error/data state.

```typescript
// client/api.ts — typed client matching the server schema
import { HttpClient } from "@effect/platform"
import { Effect } from "effect"

export const fulfillOrder = (orderId: string) =>
  HttpClient.request.post(`/orders/${orderId}/fulfill`).pipe(
    HttpClient.client.fetchOk,                        // fails on non-2xx
    Effect.asVoid
  )
```

```typescript
// client/hooks/useFulfillOrder.ts
import { useState } from "react"
import { Effect } from "effect"
import { BrowserHttpClient } from "@effect/platform-browser"
import { fulfillOrder } from "../api"

export const useFulfillOrder = () => {
  const [status, setStatus] = useState<"idle" | "loading" | "done" | "error">("idle")
  const [error, setError] = useState<string | null>(null)

  const execute = (orderId: string) => {
    setStatus("loading")
    setError(null)

    fulfillOrder(orderId).pipe(
      Effect.provide(BrowserHttpClient.layer),
      Effect.runPromise
    ).then(() => {
      setStatus("done")
    }).catch((e) => {
      setStatus("error")
      setError(String(e))
    })
  }

  return { execute, status, error }
}
```

```tsx
// client/components/OrderCard.tsx
import { useFulfillOrder } from "../hooks/useFulfillOrder"

interface Props {
  orderId: string
  currentStatus: "draft" | "fulfilled"
}

export const OrderCard = ({ orderId, currentStatus }: Props) => {
  const { execute, status, error } = useFulfillOrder()

  return (
    <div>
      <p>Order {orderId} — {currentStatus}</p>

      {currentStatus === "draft" && (
        <button
          onClick={() => execute(orderId)}
          disabled={status === "loading"}
        >
          {status === "loading" ? "Fulfilling…" : "Fulfill Order"}
        </button>
      )}

      {status === "done" && <p>Order fulfilled.</p>}
      {status === "error" && <p>Error: {error}</p>}
    </div>
  )
}
```

---

## How the layers connect — one request traced

```
POST /orders/abc-123/fulfill
        │
        │  raw HTTP, no types
        ▼
OrdersHandler
  - decodes path: { id: "abc-123" }
  - lifts: OrderId("abc-123")
  - calls: FulfillOrderService.execute(orderId)
        │
        │  OrderId (branded string)
        ▼
FulfillOrderService
  - repo.findById(orderId)          ← hits Redis first
    - cache miss → Drizzle SELECT → Postgres
    - populates Redis TTL=60s
  - order.fulfill()                 ← pure domain, no I/O
  - repo.save(order)                ← Drizzle UPSERT + Redis DEL
  - bus.publish(OrderFulfilled)     ← INSERT into outbox table
        │
        │  void
        ▼
Handler returns HTTP 200

--- 5 seconds later ---

OutboxWorker polls outbox
  - finds unpublished OrderFulfilled row
  - publishes to external broker (Kafka/SQS/log)
  - marks publishedAt = now()
```

---

## Layer responsibilities (summary)

| Layer | Allowed | NOT allowed |
|---|---|---|
| Domain | Invariants, state transitions, emit events | I/O of any kind |
| Application service | Load → call domain → save → publish | Business rules, SQL, HTTP |
| Handler | Decode HTTP, lift types, map errors | Business logic, DB access |
| Repository (DB) | Load/save aggregates, map rows | Business logic |
| Cache wrapper | Read-through, write-through, TTL | Business logic |
| Event bus (outbox) | Write event row to DB | Processing, filtering, routing |
| Outbox worker | Poll, publish, mark done | Business logic |
| Frontend hook | Run Effect, expose state | Business logic |

---

## Related

- [[full-stack-layers]]
- [[ddd-request-flow]]
- [[domain-driven-design]]
- [[ddd-aggregate-root]]
- [[ddd-domain-events]]
- [[ddd-repository-pattern]]
- [[ports-and-adapters]]

## References

- Effect source: `80-resources/effect-source/packages/effect/src/Context.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Layer.ts`
- [@effect/platform HttpApiBuilder](https://effect.website/docs/platform/http-api)
- [@effect/react](https://effect.website/docs/react)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
