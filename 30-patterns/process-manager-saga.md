---
title: Process Manager / Saga
type: pattern
created: 2026-08-05
tags: [ddd, saga, process-manager, async, effect]
effect-source: 80-resources/effect-source
---

# Process Manager / Saga

## What

A stateful coordinator for a long-running, multi-step operation that spans multiple aggregates and/or external services. Owns the state machine of the overall process — not the business rules of any individual aggregate.

Use when:
- Multiple aggregates are involved
- An external async job must be dispatched and its result written back
- Partial failure is possible and must be recovered from

## This scenario

```
1. Load multiple aggregates
2. Dispatch async job to external service
3. Store "dispatched" state (survive restarts)
4. External service completes → webhook/event arrives
5. Write result back to aggregate(s)
```

---

## State machine

```
         dispatch()
pending ──────────────▶ dispatched
                             │
              ┌──────────────┴──────────────┐
              │ success                     │ failure
              ▼                             ▼
          completed                    failed (retryCount < max)
                                            │
                                            │ retry with backoff
                                            ▼
                                       dispatched
                                            │
                                            │ retryCount >= max
                                            ▼
                                        dead_letter
```

---

## Step 1 — Model the process as an aggregate

The process manager is itself a domain object with its own state, persisted between steps. One aggregate per long-running process instance.

```typescript
// domain/pricing/ComputationProcess.ts
import { Data, Effect, Brand, Schema } from "effect"

type ComputationProcessId = string & Brand.Brand<"ComputationProcessId">
const ComputationProcessId = Brand.nominal<ComputationProcessId>()

type ComputationStatus = "pending" | "dispatched" | "completed" | "failed" | "dead_letter"

class ComputationResultReceived extends Data.TaggedClass("ComputationResultReceived")<{
  readonly processId: ComputationProcessId
  readonly result: number
}> {}

class ComputationDispatched extends Data.TaggedClass("ComputationDispatched")<{
  readonly processId: ComputationProcessId
  readonly jobId: string
}> {}

type ProcessEvent = ComputationResultReceived | ComputationDispatched

export class ComputationProcess {
  private _events: ProcessEvent[] = []

  constructor(
    readonly id: ComputationProcessId,
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    private _status: ComputationStatus,
    private _jobId: string | null,
    private _result: number | null,
    private _retryCount: number,
    readonly MAX_RETRIES = 3,
  ) {}

  get status() { return this._status }
  get jobId() { return this._jobId }
  get result() { return this._result }
  get retryCount() { return this._retryCount }

  // Called when job is dispatched to external service
  markDispatched(jobId: string): Effect.Effect<void, ProcessAlreadyCompleted> {
    return Effect.gen(function* (this: ComputationProcess) {
      if (this._status === "completed")
        yield* Effect.fail(new ProcessAlreadyCompleted({ id: this.id }))
      this._jobId = jobId
      this._status = "dispatched"
      this._events.push(new ComputationDispatched({ processId: this.id, jobId }))
    }.bind(this))
  }

  // Called when external service responds (webhook/event)
  receiveResult(result: number): Effect.Effect<void, InvalidProcessState> {
    return Effect.gen(function* (this: ComputationProcess) {
      if (this._status !== "dispatched")
        yield* Effect.fail(new InvalidProcessState({ id: this.id, status: this._status }))
      this._result = result
      this._status = "completed"
      this._events.push(new ComputationResultReceived({ processId: this.id, result }))
    }.bind(this))
  }

  // Called when external service call fails
  markFailed(): Effect.Effect<void, never> {
    return Effect.sync(() => {
      this._retryCount += 1
      this._status = this._retryCount >= this.MAX_RETRIES ? "dead_letter" : "failed"
    })
  }

  canRetry(): boolean {
    return this._status === "failed" && this._retryCount < this.MAX_RETRIES
  }

  pullEvents(): ProcessEvent[] {
    const events = [...this._events]
    this._events = []
    return events
  }
}
```

---

## Step 2 — External service port

The domain declares what it needs from the external service — infrastructure implements it.

```typescript
// domain/pricing/ExternalComputationService.ts
import { Context, Effect } from "effect"

export interface ExternalComputationService {
  readonly dispatch: (input: {
    orderId: OrderId
    customerId: CustomerId
  }) => Effect.Effect<{ jobId: string }, ExternalServiceError>
}

export class ExternalComputationService extends Context.Service<ExternalComputationService>()(
  "ExternalComputationService"
) {}
```

---

## Step 3 — Application service: dispatch phase

Loads the aggregates needed, creates the process, dispatches the job, persists state.

```typescript
// application/DispatchComputationService.ts
import { Context, Effect, Layer, Schedule } from "effect"
import { ComputationProcess } from "../domain/pricing/ComputationProcess"
import { ComputationProcessRepository } from "../domain/pricing/ComputationProcessRepository"
import { ExternalComputationService } from "../domain/pricing/ExternalComputationService"
import { OrderRepository } from "../domain/order/OrderRepository"
import { CustomerRepository } from "../domain/customer/CustomerRepository"

// --- Service interface ---

export interface DispatchComputationService {
  readonly execute: (
    orderId: OrderId,
    customerId: CustomerId
  ) => Effect.Effect<
    ComputationProcessId,
    OrderNotFound | CustomerNotFound | ExternalServiceError | PersistenceError
  >
}

export class DispatchComputationService extends Context.Service<DispatchComputationService>()(
  "DispatchComputationService"
) {}

// --- Layer (implementation) ---

export const DispatchComputationServiceLive = Layer.effect(
  DispatchComputationService,
  Effect.gen(function* () {
    const orderRepo = yield* OrderRepository
    const customerRepo = yield* CustomerRepository
    const processRepo = yield* ComputationProcessRepository
    const externalSvc = yield* ExternalComputationService

    return DispatchComputationService.of({
      execute: (orderId, customerId) =>
        Effect.gen(function* () {
          // 1. Load aggregates
          const order = yield* orderRepo.findById(orderId)
          const customer = yield* customerRepo.findById(customerId)

          // 2. Create process — persisted immediately, survives restarts
          const process = new ComputationProcess(
            ComputationProcessId(crypto.randomUUID()),
            order.id,
            customer.id,
            "pending",
            null,
            null,
            0,
          )
          yield* processRepo.save(process)

          // 3. Dispatch with retry + exponential backoff
          const { jobId } = yield* externalSvc
            .dispatch({ orderId, customerId })
            .pipe(
              Effect.retry(
                Schedule.exponential("100 millis").pipe(
                  Schedule.intersect(Schedule.recurs(3))
                )
              ),
              Effect.tapError(() => process.markFailed()),
            )

          // 4. Record dispatched state
          yield* process.markDispatched(jobId)
          yield* processRepo.save(process)

          return process.id
        }),
    })
  })
)
```

---

## Step 4 — Handle completion (webhook / event handler)

When the external service responds, this is the entry point. Same structure as a regular application service — just triggered by an event rather than an HTTP request.

```typescript
// application/HandleComputationResultService.ts
import { Context, Effect, Layer } from "effect"
import { ComputationProcessRepository } from "../domain/pricing/ComputationProcessRepository"
import { OrderRepository } from "../domain/order/OrderRepository"
import { EventBus } from "../infrastructure/EventBus"

// --- Service interface ---

export interface HandleComputationResultService {
  readonly execute: (
    jobId: string,
    result: number
  ) => Effect.Effect<
    void,
    ProcessNotFound | InvalidProcessState | OrderNotFound | PersistenceError
  >
}

export class HandleComputationResultService extends Context.Service<HandleComputationResultService>()(
  "HandleComputationResultService"
) {}

// --- Layer (implementation) ---

export const HandleComputationResultServiceLive = Layer.effect(
  HandleComputationResultService,
  Effect.gen(function* () {
    const processRepo = yield* ComputationProcessRepository
    const orderRepo = yield* OrderRepository
    const bus = yield* EventBus

    return HandleComputationResultService.of({
      execute: (jobId, result) =>
        Effect.gen(function* () {
          // 1. Find which process this result belongs to
          const process = yield* processRepo.findByJobId(jobId)

          // 2. Enforce state machine — only accepts result if "dispatched"
          yield* process.receiveResult(result)

          // 3. Load target aggregate and apply result
          const order = yield* orderRepo.findById(process.orderId)
          yield* order.applyComputedDiscount(result)

          // 4. Persist both
          yield* processRepo.save(process)
          yield* orderRepo.save(order)

          // 5. Publish events after persistence
          const events = [...process.pullEvents(), ...order.pullEvents()]
          yield* Effect.forEach(events, bus.publish, { discard: true })
        }),
    })
  })
)
```

---

## Step 5 — Retry worker (for failed processes)

A background worker periodically picks up failed processes and retries dispatch. Modeled as a `Context.Service` so it's injectable and testable like everything else.

```typescript
// infrastructure/workers/ComputationRetryWorker.ts
import { Context, Effect, Layer, Schedule, Duration, Stream } from "effect"
import { ComputationProcessRepository } from "../../domain/pricing/ComputationProcessRepository"
import { DispatchComputationService } from "../../application/DispatchComputationService"

// --- Service interface ---

export interface ComputationRetryWorker {
  readonly start: Effect.Effect<never>  // runs forever
}

export class ComputationRetryWorker extends Context.Service<ComputationRetryWorker>()(
  "ComputationRetryWorker"
) {}

// --- Layer (implementation) ---

export const ComputationRetryWorkerLive = Layer.effect(
  ComputationRetryWorker,
  Effect.gen(function* () {
    const repo = yield* ComputationProcessRepository
    const dispatcher = yield* DispatchComputationService

    const tick = Effect.gen(function* () {
      const failed = yield* repo.findRetryable()  // status = "failed", retryCount < MAX

      yield* Effect.forEach(
        failed,
        (process) =>
          dispatcher.execute(process.orderId, process.customerId).pipe(
            Effect.catchAll((e) => Effect.logError("Retry failed", e))  // don't crash the worker
          ),
        { concurrency: 5 }
      )
    })

    return ComputationRetryWorker.of({
      start: tick.pipe(
        Effect.repeat(Schedule.spaced(Duration.seconds(30))),
        Effect.asVoid,
      ),
    })
  })
)
```

Started at the app boundary alongside the HTTP server:

```typescript
// main.ts
const AppLive = Layer.mergeAll(
  HttpServerLive,
  Layer.launch(
    Layer.effect(
      Layer.succeed(ComputationRetryWorker, /* ... */),
      Effect.gen(function* () {
        const worker = yield* ComputationRetryWorker
        yield* worker.start
      })
    )
  )
)
```

---

## Where things live

```
domain/
  pricing/
    ComputationProcess.ts          ← process aggregate (state machine)
    ComputationProcessRepository.ts ← port
    ExternalComputationService.ts   ← port

application/
  DispatchComputationService.ts    ← dispatch phase handler
  HandleComputationResultService.ts ← completion phase handler

infrastructure/
  persistence/
    DrizzleComputationProcessRepository.ts
  external/
    HttpExternalComputationService.ts  ← calls the real external API
  workers/
    ComputationRetryWorker.ts
```

---

## Key rules

- **Process is a first-class aggregate** — persisted immediately, survives restarts and crashes
- **Idempotency** — `receiveResult` must guard against duplicate webhook deliveries (state check: only accept if `dispatched`)
- **Retry state lives on the aggregate** — `retryCount`, `MAX_RETRIES`, `canRetry()` — not in the worker
- **Write result via the aggregate** — `order.applyComputedDiscount(result)` enforces invariants; don't write directly to DB
- **Never hold aggregates in memory across async gaps** — reload from DB when the webhook arrives

---

## Related

- [[domain-driven-design]]
- [[ddd-aggregate-root]] — process manager is itself an aggregate
- [[ddd-domain-events]] — completion triggers an event; choreography alternative
- [[ddd-repository-pattern]]
- [[unit-of-work-pattern]] — wrap steps 2+3 in `withTransaction` if atomicity needed
- [[ddd-request-flow]]

## References

- Evans, *DDD* — process manager / saga
- Vernon, *IDDD* ch. 10 — aggregates and domain events
- Effect source: `80-resources/effect-source/packages/effect/src/Schedule.ts`
