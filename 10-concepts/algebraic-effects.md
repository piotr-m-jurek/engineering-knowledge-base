---
title: Algebraic Effects Systems
type: concept
created: 2026-08-11
tags: [effect, functional-programming, concurrency, typescript, scala, ocaml, zio, architecture]
effect-source: 80-resources/effect-source
---

# Algebraic Effects Systems

## What

A type-safe way to **describe computations without executing them** — separating *what* a program does from *how* it is run.

The runtime (interpreter) decides scheduling, error routing, concurrency, resource cleanup, and tracing. The program is a pure data structure handed to it.

The word *algebraic* comes from Plotkin & Power (2001): effects form a free monad over an algebra of operations. Each `yield*` (or `perform`) is a request sent upward to a handler. The handler can:
1. Resume with a value (normal execution, dependency injection)
2. Resume with a different value (error recovery)
3. Not resume (interruption, resource cleanup)

Operations compose with consistent laws — you can reason about them locally, the same way arithmetic expressions compose.

---

## The core type

```typescript
Effect<A, E, R>
//     ^  ^  ^
//     |  |  └── Requirements — services this effect needs
//     |  └───── Error channel — typed, expected failures
//     └──────── Success value
```

Creating an `Effect` does **not** execute it. It builds a value that is composed, provided with services, and handed to the runtime. Execution is deferred until `Effect.runMain` / `Effect.runPromise`.

---

## Why it exists — what it replaces

| Problem with plain async/Promise | Effect solution |
|---|---|
| Errors disappear into `any` / `unknown` | `E` channel — compiler-checked union |
| Dependencies smuggled via closure or global | `R` channel — declared, injected |
| No safe cancellation | Structured fiber interruption |
| Resource cleanup fragile (`try/finally`) | `Scope` — guaranteed finalizers |
| `Promise` fires immediately, no composition | `Effect` is lazy, composable before running |
| Retry, timeout, tracing: manual boilerplate | Combinators: `Effect.retry`, `Effect.timeout`, `Effect.withSpan` |

---

## The three channels in detail

### `A` — Success value

What the computation produces when it succeeds. Equivalent to a return type.

### `E` — Error channel

A discriminated union of **expected, typed failures**. Not a catch-all `Error`. The compiler tracks what errors remain unhandled.

```typescript
// ai-docs/src/01_effect/04_errors/01_error-handling.ts
import { Effect, Schema } from "effect"

class ParseError extends Schema.TaggedError<ParseError>()("ParseError", {
  input: Schema.String,
  message: Schema.String
}) {}

class ReservedPortError extends Schema.TaggedError<ReservedPortError>()("ReservedPortError", {
  port: Schema.Int
}) {}

declare const loadPort: (input: string) => Effect.Effect<number, ParseError | ReservedPortError>

// After catchTag, E = never — compiler confirms all errors handled
const recovered = loadPort("80").pipe(
  Effect.catchTag(["ParseError", "ReservedPortError"], (_) => Effect.succeed(3000))
)
```

### `R` — Requirements (services)

Accumulates required services across the call tree. The compiler refuses to run a program where `R ≠ never`. You must provide all declared dependencies before handing the effect to the runtime.

```typescript
// R = Database — TypeScript knows this needs Database
const getUser = Effect.fn("getUser")(function*(id: string) {
  const row = yield* Database.query(`SELECT * FROM users WHERE id = $1`)
  return row as User
})
// Type: Effect<User, DatabaseError, Database>
```

---

## Services and Layers

Services declare what a component needs; Layers describe how to build them.

```typescript
// ai-docs/src/01_effect/03_services/01_service.ts
import { Context, Effect, Layer, Schema } from "effect"

class Database extends Context.Service<Database, {
  query(sql: string): Effect.Effect<Array<unknown>, DatabaseError>
}>()(
  "myapp/db/Database"   // unique tag — the DI key
) {
  static readonly layer = Layer.effect(
    Database,
    Effect.gen(function*() {
      const query = Effect.fn("Database.query")(function*(sql: string) {
        yield* Effect.log("Executing SQL query:", sql)
        return [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }]
      })
      return Database.of({ query })
    })
  )
}
```

`Layer` is composable — it forms a dependency graph, type-checked:

```typescript
const AppLayer = Database.layer.pipe(
  Layer.provide(PgPoolLayer),    // Database needs a pool
  Layer.provide(ConfigLayer)     // pool needs config
)
// The full wiring, verified by the compiler
```

**Testing** = swap the `Layer`. The program is unchanged:

```typescript
const test = program.pipe(Effect.provide(FakeDatabase.layer))
const prod = program.pipe(Effect.provide(Database.layer))
```

---

## Writing effects

Prefer `Effect.gen` for imperative style and `Effect.fn` for named functions.

```typescript
// Effect.gen — looks like async/await
Effect.gen(function*() {
  yield* Effect.log("Starting...")
  const user = yield* getUser("123")   // yield* unwraps the result
  return user
})

// Effect.fn — named function returning an Effect (adds span + stack trace)
export const processFile = Effect.fn("processFile")(
  function*(path: string) {
    yield* Effect.log("Reading:", path)
    return yield* new FileProcessingError({ message: "Failed" })
  },
  Effect.catch((error) => Effect.logError(`Error: ${error}`))
)
```

Source: `80-resources/effect-source/ai-docs/src/01_effect/01_basics/01_effect-gen.ts`

---

## Fibers — the concurrency model

A **fiber** is a lightweight cooperative thread managed by the runtime. Not an OS thread — thousands can run concurrently in a single process.

```typescript
// packages/effect/src/Fiber.ts
// "A runtime fiber is a lightweight thread that executes Effects.
//  Fibers are the unit of concurrency in Effect."

const program = Effect.gen(function*() {
  const fiber = yield* Effect.forkChild(heavyWork)  // non-blocking fork
  yield* doOtherWork()                              // runs concurrently
  const result = yield* Fiber.await(fiber)          // structured join
})
```

Key properties:
- **Structured** — parent fiber owns children; interrupt parent → children interrupted
- **Typed** — `Fiber<A, E>` carries success/error types
- **Safe** — interruption triggers finalizers, no resource leaks

---

## Scope — resource safety

A `Scope` is a lifetime boundary. Finalizers registered inside are guaranteed to run on close — regardless of success, failure, or interruption.

```typescript
// packages/effect/src/Scope.ts
// "A scope is a lifetime boundary."

const withFile = Effect.acquireRelease(
  Effect.sync(() => fs.openSync("data.txt", "r")),   // acquire
  (fd) => Effect.sync(() => fs.closeSync(fd))         // release — always runs
)
// No try/finally. Cleanup is guaranteed even under fiber interrupt.
```

`Scope` is threaded through `R` automatically when you use `Effect.scoped`. The runtime closes it on exit.

---

## Running an effect

An effect is just a description until you hand it to the runtime. `R` must be `never` at that point.

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Layer } from "effect"

const program = Layer.launch(AppWorker)

NodeRuntime.runMain(program, { disableErrorReporting: true })
// Installs SIGINT/SIGTERM handlers, graceful fiber shutdown
```

Source: `80-resources/effect-source/ai-docs/src/01_effect/06_running/10_run-main.ts`

---

## Comparison across systems

| Concept | Effect (TS) | ZIO (Scala) | OCaml Multicore |
|---|---|---|---|
| Effect type | `Effect<A, E, R>` | `ZIO[R, E, A]` | `'a Effect.t` + handlers |
| Services / DI | `Context.Service` + `Layer` | `ZLayer` + `ZEnvironment` | Effect handlers as dynamic bind |
| Error channel | `E`, `Schema.TaggedError` | `E`, `case class` with `ZIO.fail` |  Exception-like algebraic effects |
| Fibers | `Effect.fork*` | `ZIO.fork*` | `Eio.Fiber.fork` / `Domain.spawn` |
| Resources | `Scope`, `acquireRelease` | `ZManaged` / `Scope` | `Fun.finally` / `Resource` |
| Interpretation | `Effect.runMain` | `ZIOApp.run` | Effect handler `match`/`try…with` |
| Language support | Emulated (generator coroutines) | Emulated (flatMap chains) | First-class language primitive |

**OCaml Multicore** is the purest form: `perform` and `continue` are language keywords. The handler is installed with `match_with` or `Effect.Deep.match`. No emulation — the runtime suspends the stack at `perform` and passes control to the handler, which can resume it with a value.

ZIO and Effect TS **simulate** the same model using coroutines / monadic chaining. The programming model is identical; only handler performance and expressiveness differ.

---

## Architectural implication — `R` as a capability system

The `R` channel enforces **ports-and-adapters** at the type level. Domain code lists what it needs; infrastructure provides it. The compiler rejects programs with unresolved dependencies.

```typescript
// Domain layer — pure descriptions, no concrete implementations
const placeOrder = Effect.fn("placeOrder")(function*(cmd: PlaceOrderCmd) {
  const inventory = yield* Inventory   // R: Inventory
  const payments  = yield* Payments    // R: Payments
  // business logic here — no knowledge of Postgres, Stripe, etc.
})
// Type: Effect<Order, DomainError, Inventory | Payments>

// Infrastructure layer wires it — domain code unchanged
const AppLayer = Layer.mergeAll(
  PostgresInventory.layer,
  StripePayments.layer
)
```

> The domain layer has `R = never` after providing infrastructure. You **cannot** accidentally call domain code with unresolved infrastructure dependencies — the compiler refuses.

→ See [[ports-and-adapters]]  
→ See [[70-projects/ddd-finance-app]] for a full worked example

---

## Composability — why it's algebraic

Every combinator transforms `Effect → Effect`. No hidden state, no callbacks mutating shared mutable things. Reasoning is local:

```typescript
const program = Effect.all([fetchUser(1), fetchUser(2)], { concurrency: 2 }).pipe(
  Effect.retry({ times: 3 }),
  Effect.timeout("5 seconds"),
  Effect.withSpan("fetch-users")
)
```

Each combinator adds one concern. Remove one — the others still work. This is the algebraic property: operations follow consistent laws that let you compose and reason independently.

---

## Glossary

| Term | Meaning |
|---|---|
| **Effect** | Lazy description of a computation — not executed until handed to the runtime |
| **Fiber** | Lightweight cooperative thread managed by the runtime |
| **Layer** | Blueprint for building and wiring services; composable DI container |
| **Scope** | Lifetime boundary — resources acquired inside are released when it closes |
| **Context / Service** | Named, typed dependency tracked in `R` |
| **Algebraic** | Follows laws that let operations compose and be reasoned about locally |
| **Handler** | Runtime component interpreting effect requests (resume, abort, redirect) |
| **`R = never`** | All dependencies satisfied — program is runnable |
| **`perform`** | OCaml keyword: suspend current stack, send request to handler |
| **`continue`** | OCaml keyword: resume a suspended stack with a value |

---

## Related

- [[ports-and-adapters]] — `R` as a type-level enforcement of port/adapter separation
- [[domain-driven-design]] — domain services use `R` to declare infrastructure needs
- [[ddd-repository-pattern]] — classic use of `Context.Service` + `Layer` swap
- [[durable-execution]] — workflows as effects with persisted interpretation
- [[70-projects/ddd-finance-app]] — full reference implementation with Effect v4

## References

- Plotkin & Power, *Adequacy for Algebraic Effects* (2001) — theoretical origin
- Effect (TS) source: `80-resources/effect-source/packages/effect/src/Effect.ts`
- Effect (TS) services: `80-resources/effect-source/ai-docs/src/01_effect/03_services/01_service.ts`
- Effect (TS) errors: `80-resources/effect-source/ai-docs/src/01_effect/04_errors/01_error-handling.ts`
- Effect (TS) fibers: `80-resources/effect-source/packages/effect/src/Fiber.ts`
- Effect (TS) scope: `80-resources/effect-source/packages/effect/src/Scope.ts`
- OCaml 5 effects: https://v2.ocaml.org/api/compiledfiles/Effect.html
- ZIO docs: https://zio.dev/reference/core/zio/
