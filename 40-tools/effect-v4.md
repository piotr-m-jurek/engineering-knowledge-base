---
title: Effect v4
type: tool
created: 2026-08-11
tags: [effect, typescript, functional, concurrency, dependency-injection]
website: https://effect.website
---

# Effect v4

## What

Effect is a TypeScript library for building production-grade applications. It provides typed errors, dependency injection via `Context`/`Layer`, structured concurrency, resource management, observability, and composable streams — all with full type inference.

v4 is a breaking rework of v3. Many APIs moved, renamed, or were removed.

---

## Source resource

Checked in as a git submodule at `80-resources/effect-source/`.

| What to look up | Where |
|---|---|
| Full API overview | `80-resources/effect-source/LLMS.md` |
| v3 → v4 migration | `80-resources/effect-source/MIGRATION.md` |
| Runnable examples | `80-resources/effect-source/ai-docs/` |
| Cookbooks | `80-resources/effect-source/cookbooks/` |
| Core module source | `80-resources/effect-source/packages/effect/src/` |
| HTTP (unstable) | `80-resources/effect-source/packages/effect/src/unstable/http/` |

When writing Effect code: check `LLMS.md` first, then `ai-docs/` for a working example, fall back to package source only for edge cases. Always verify import paths — v4 moves many things.

---

## Key commands / API

| Concept | v3 | v4 |
|---|---|---|
| Service tag | `Context.Tag` | `Context.Service` |
| Effect tag | `Effect.Tag` | `Context.Service` |
| Scoped layer | `Layer.scoped` | `Layer.effect` (handles Scope) |
| HTTP imports | `@effect/platform` | `effect/unstable/http` |
| Either | `Either` | `Result` |
| Service accessor | service proxy | `yield* Service` or `Service.use(fn)` |
| Default layer | `.Default` | `.layer` |

---

## Service definition pattern

```typescript
import { Context, Effect, Layer } from "effect"

class MyService extends Context.Service<MyService>()<
  "myapp/MyService",
  {
    doThing(x: string): Effect.Effect<string, MyError>
  }
>()("myapp/MyService") {
  static readonly layer = Layer.effect(
    MyService,
    Effect.gen(function* () {
      return MyService.of({
        doThing: Effect.fn("MyService.doThing")(function* (x) {
          return yield* Effect.succeed("done")
        }),
      })
    })
  )
}
```

---

## Key idioms

**Named effectful functions** — use `Effect.fn("name")(function*() {...})`. Adds spans and stack traces automatically.

```typescript
const fetchUser = Effect.fn("UserService.fetchUser")(function* (id: string) {
  const db = yield* Database
  return yield* db.findUser(id)
})
```

**Yield errors directly** — domain errors extend `Schema.TaggedError` and are yieldable inside `Effect.gen`. No `Effect.fail` wrapper needed.

```typescript
class NotFoundError extends Schema.TaggedError<NotFoundError>()(
  "NotFoundError",
  { id: Schema.String }
) {}

const getUser = Effect.fn("getUser")(function* (id: string) {
  const user = yield* db.find(id)
  if (!user) yield* new NotFoundError({ id })  // direct yield
  return user
})
```

**Using a service** — yield the service class to get its implementation.

```typescript
const program = Effect.gen(function* () {
  const svc = yield* MyService
  const result = yield* svc.doThing("hello")
  return result
})
```

---

## Gotchas

- `Context.Tag` removed — compiler error will say class doesn't extend correct base. Use `Context.Service`.
- `Layer.scoped` removed — just use `Layer.effect`; it handles `Scope` automatically when the effect acquires resources.
- `@effect/platform` HTTP merged into core — import from `effect/unstable/http`, not `@effect/platform/HttpClient`.
- `Either` renamed to `Result` — search/replace in migrations.
- Service proxy accessor removed — `yield* MyService.someMethod(...)` no longer works. `yield* MyService` first, then call methods on the result.
- `.Default` layer convention changed to `.layer` — update all `provide(MyService.Default)` to `provide(MyService.layer)`.

---

## Alternatives

| Library | Tradeoff |
|---|---|
| `fp-ts` | Predecessor, functional style, less ergonomic DI, no built-in concurrency |
| `neverthrow` | Typed errors only, no DI or concurrency story |
| Raw `Promise` + `zod` | Simple, no structured concurrency or resource management |

---

## Related

- `[[20-languages/typescript]]`
- `[[30-patterns/ports-and-adapters]]` — Effect `Layer` is the canonical way to implement this pattern
- `[[10-concepts/durable-execution]]` — Effect Workflows use the same replay model
- `[[70-projects/ddd-finance-app]]` — reference project using Effect v4 throughout

## References

- Effect LLMS.md — `80-resources/effect-source/LLMS.md`
- Effect Migration Guide — `80-resources/effect-source/MIGRATION.md`
- https://effect.website/docs
