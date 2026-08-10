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

## Layer 6 — Infrastructure: Event Bus

Two real-world options. Both use the **Transactional Outbox** pattern as the bridge between the DB write and the broker — the `EventBus` port implementation writes to an `outbox` table in the same DB transaction; a background worker picks up rows and forwards to the broker.

```
repo.save(order)            ← Postgres UPSERT
bus.publish(OrderFulfilled) ← Postgres INSERT into outbox (same tx)
        │
        │  (async, outbox worker)
        ▼
BullMQ queue  OR  RabbitMQ exchange
        │
        ▼
Consumer worker processes the job/message
```

### Outbox table (shared by both)

```typescript
// infrastructure/outbox/schema.ts
import { pgTable, text, timestamp, jsonb } from "drizzle-orm/pg-core"

export const outbox = pgTable("outbox", {
  id: text("id").primaryKey(),
  type: text("type").notNull(),           // event _tag e.g. "OrderFulfilled"
  payload: jsonb("payload").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  publishedAt: timestamp("published_at"), // null = pending
})
```

```typescript
// infrastructure/outbox/OutboxEventBus.ts — same for both brokers
import { Effect, Layer } from "effect"
import { EventBus } from "../../domain/order/EventBus"
import { PersistenceError } from "../../domain/order/errors"
import { db } from "../persistence/db"
import { outbox } from "./schema"

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

---

### Option A — BullMQ (Redis-backed)

BullMQ stores jobs in Redis. Each event type becomes a named queue. Workers pull jobs and process them with automatic retries, backoff, and dead-letter support.

**When to use:** single service or small multi-service setups. No separate broker process — Redis is already there for the cache layer.

```
npm install bullmq
```

```typescript
// infrastructure/messaging/bullmq/queues.ts
import { Queue } from "bullmq"

const connection = { host: process.env.REDIS_HOST, port: 6379 }

// One queue per event type (or one shared queue — your choice)
export const orderFulfilledQueue = new Queue("OrderFulfilled", { connection })
```

```typescript
// infrastructure/messaging/bullmq/OutboxWorker.ts
import { Effect, Schedule, Layer } from "effect"
import { isNull, eq } from "drizzle-orm"
import { db } from "../../persistence/db"
import { outbox } from "../schema"
import { orderFulfilledQueue } from "./queues"

const forwardToQueue = (row: typeof outbox.$inferSelect) =>
  Effect.tryPromise({
    try: async () => {
      // Add job to the BullMQ queue — Redis-backed, durable
      await orderFulfilledQueue.add(
        row.type,          // job name
        row.payload,       // job data
        {
          jobId: row.id,   // idempotent: re-enqueue same row → same jobId, no duplicate
          attempts: 5,
          backoff: { type: "exponential", delay: 1000 },
        }
      )
      // Mark as published only after successful enqueue
      await db.update(outbox)
        .set({ publishedAt: new Date() })
        .where(eq(outbox.id, row.id))
    },
    catch: (e) => new Error(String(e)),
  })

export const BullMQOutboxWorkerLive = Layer.scopedDiscard(
  Effect.gen(function* () {
    const poll = Effect.tryPromise({
      try: () =>
        db.select().from(outbox)
          .where(isNull(outbox.publishedAt))
          .limit(50),
      catch: (e) => new Error(String(e)),
    }).pipe(
      Effect.flatMap((rows) =>
        Effect.forEach(rows, forwardToQueue, { concurrency: 5, discard: true })
      )
    )

    yield* Effect.repeat(poll, Schedule.spaced("2 seconds")).pipe(
      Effect.forkScoped
    )
  })
)
```

```typescript
// infrastructure/messaging/bullmq/OrderFulfilledWorker.ts — consumer
import { Worker } from "bullmq"
import { Effect, Layer } from "effect"

// What to do when OrderFulfilled lands in the queue
const processOrderFulfilled = async (job: { data: { orderId: string; customerId: string } }) => {
  const { orderId, customerId } = job.data
  console.log(`[bullmq] processing OrderFulfilled orderId=${orderId} customerId=${customerId}`)
  // e.g.: trigger email, update analytics, notify inventory service
}

export const OrderFulfilledWorkerLive = Layer.scopedDiscard(
  Effect.acquireRelease(
    Effect.sync(() =>
      new Worker(
        "OrderFulfilled",
        processOrderFulfilled,
        {
          connection: { host: process.env.REDIS_HOST, port: 6379 },
          concurrency: 10,
        }
      )
    ),
    (worker) => Effect.promise(() => worker.close())  // graceful shutdown
  )
)
```

**Retry flow:** BullMQ retries failed jobs up to `attempts` times with exponential backoff. Failed jobs past the limit go to the dead-letter queue (`failed` set in Redis) for inspection.

---

### Option B — RabbitMQ (AMQP)

RabbitMQ routes messages via exchanges. One exchange per domain + binding keys per event type. Multiple consumers can subscribe independently (fan-out). Better for cross-service integration where consumers are in different codebases.

**When to use:** multiple services need to react to the same event independently, or you need complex routing (topic, header-based).

```
npm install amqplib
npm install -D @types/amqplib
```

```typescript
// infrastructure/messaging/rabbitmq/connection.ts
import amqp from "amqplib"

let _channel: amqp.Channel | null = null

export const getChannel = async (): Promise<amqp.Channel> => {
  if (_channel) return _channel
  const conn = await amqp.connect(process.env.RABBITMQ_URL ?? "amqp://localhost")
  _channel = await conn.createChannel()

  // Declare durable exchange — survives broker restart
  await _channel.assertExchange("orders", "topic", { durable: true })

  return _channel
}
```

```typescript
// infrastructure/messaging/rabbitmq/OutboxWorker.ts
import { Effect, Schedule, Layer } from "effect"
import { isNull, eq } from "drizzle-orm"
import { db } from "../../persistence/db"
import { outbox } from "../schema"
import { getChannel } from "./connection"

const forwardToExchange = (row: typeof outbox.$inferSelect) =>
  Effect.tryPromise({
    try: async () => {
      const channel = await getChannel()

      // Routing key = event type, e.g. "OrderFulfilled"
      // Consumers bind queues to this exchange with matching routing key patterns
      channel.publish(
        "orders",           // exchange name
        row.type,           // routing key: "OrderFulfilled"
        Buffer.from(JSON.stringify(row.payload)),
        {
          persistent: true,           // survives RabbitMQ restart
          messageId: row.id,          // idempotency key for consumers
          contentType: "application/json",
          timestamp: Date.now(),
        }
      )

      await db.update(outbox)
        .set({ publishedAt: new Date() })
        .where(eq(outbox.id, row.id))
    },
    catch: (e) => new Error(String(e)),
  })

export const RabbitMQOutboxWorkerLive = Layer.scopedDiscard(
  Effect.gen(function* () {
    const poll = Effect.tryPromise({
      try: () =>
        db.select().from(outbox)
          .where(isNull(outbox.publishedAt))
          .limit(50),
      catch: (e) => new Error(String(e)),
    }).pipe(
      Effect.flatMap((rows) =>
        Effect.forEach(rows, forwardToExchange, { concurrency: 5, discard: true })
      )
    )

    yield* Effect.repeat(poll, Schedule.spaced("2 seconds")).pipe(
      Effect.forkScoped
    )
  })
)
```

```typescript
// infrastructure/messaging/rabbitmq/OrderFulfilledConsumer.ts — consumer
import amqp from "amqplib"
import { Effect, Layer } from "effect"
import { getChannel } from "./connection"

export const OrderFulfilledConsumerLive = Layer.scopedDiscard(
  Effect.promise(async () => {
    const channel = await getChannel()

    // Each service declares its own durable queue and binds to the exchange
    const { queue } = await channel.assertQueue("notification-service.order-fulfilled", {
      durable: true,        // survives restart
      arguments: {
        "x-dead-letter-exchange": "orders.dlx",  // failed messages → DLX
      },
    })

    await channel.bindQueue(queue, "orders", "OrderFulfilled")

    // prefetch = process one message at a time per consumer instance
    channel.prefetch(1)

    channel.consume(queue, async (msg) => {
      if (!msg) return

      try {
        const event = JSON.parse(msg.content.toString())
        console.log(`[rabbitmq] OrderFulfilled orderId=${event.orderId}`)
        // e.g.: send email, update read model, notify inventory
        channel.ack(msg)           // remove from queue: processing succeeded
      } catch (e) {
        console.error("[rabbitmq] processing failed", e)
        channel.nack(msg, false, false)  // false = don't requeue → goes to DLX
      }
    })
  })
)
```

**Retry flow:** failed messages go to the dead-letter exchange (`orders.dlx`). A separate queue bound to the DLX holds them for inspection or scheduled retry.

---

### Comparison

| | BullMQ | RabbitMQ |
|---|---|---|
| Infrastructure | Redis (already needed for cache) | Separate RabbitMQ process |
| Fan-out | Manual (multiple queues) | Native (exchange bindings) |
| Routing | Queue-per-type | Exchange + routing keys |
| Retry / backoff | Built-in, configurable | Via DLX + TTL |
| Dead-letter | `failed` set in Redis | Dead-letter exchange |
| Dashboard | Bull Board / Arena | RabbitMQ Management UI |
| Best for | Single service, simple pub/sub | Multi-service fan-out, complex routing |

The `EventBus` port and `OutboxEventBus` implementation are **identical** for both — only the outbox worker and consumer change.

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
// Pick one:
import { BullMQOutboxWorkerLive, OrderFulfilledWorkerLive } from "./infrastructure/messaging/bullmq/OutboxWorker"
// import { RabbitMQOutboxWorkerLive, OrderFulfilledConsumerLive } from "./infrastructure/messaging/rabbitmq/OutboxWorker"

const HttpLive = HttpRouter.serve(
  HttpApiBuilder.router(Api, [OrdersHandlerLive])
).pipe(
  Layer.provide(OrdersHandlerLive),
  Layer.provide(FulfillOrderServiceLive),
  Layer.provide(OrderRepositoryLive),      // Cache wrapping Drizzle
  Layer.provide(OutboxEventBusLive),       // writes to outbox table
  Layer.provideMerge(NodeHttpServer.layer({ port: 3000 }))
)

const AppLive = Layer.mergeAll(
  HttpLive,
  BullMQOutboxWorkerLive,    // polls outbox → enqueues to Redis
  OrderFulfilledWorkerLive,  // pulls from BullMQ queue → processes
)

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

--- 2 seconds later ---

BullMQOutboxWorker polls outbox
  - finds unpublished OrderFulfilled row
  - orderFulfilledQueue.add(payload, { jobId: row.id })  ← Redis
  - marks publishedAt = now()

--- immediately (BullMQ picks up job) ---

OrderFulfilledWorker.process(job)
  - sends email / updates read model / notifies inventory
  - job.ack() → removed from queue

  (on failure → retry with backoff → dead-letter after N attempts)
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
| Outbox worker | Poll outbox → enqueue to BullMQ/RabbitMQ | Business logic |
| Queue consumer | Pull job/message → process side effect | Business logic |
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
- [BullMQ docs](https://docs.bullmq.io)
- [RabbitMQ AMQP concepts](https://www.rabbitmq.com/tutorials/amqp-concepts)
