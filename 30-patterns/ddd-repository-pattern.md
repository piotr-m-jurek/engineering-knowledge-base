---
title: DDD Repository Pattern
type: pattern
created: 2026-08-05
tags: [ddd, architecture, persistence, ports-and-adapters, effect]
effect-source: 80-resources/effect-source
---

# DDD Repository Pattern

## Intent

Abstraction over persistence — the domain doesn't know storage exists. Interface defined in domain layer, implementation in infrastructure layer.

## Structure

```
domain/
  user/
    User.ts              ← aggregate root
    UserRepository.ts    ← interface (port)

infrastructure/
  persistence/
    DrizzleUserRepository.ts  ← implementation (adapter)
```

## Errors (Effect `Data.TaggedError`)

```typescript
import { Data } from "effect"

export class UserNotFound extends Data.TaggedError("UserNotFound")<{
  readonly id: UserId
}> {}

export class PersistenceError extends Data.TaggedError("PersistenceError")<{
  readonly cause: unknown
}> {}
```

`Data.TaggedError` gives structural equality, a `_tag` for `Effect.catchTag`, and makes errors yieldable in generators.

## Value Objects (Effect `Brand`)

```typescript
import { Brand } from "effect"

export type UserId = string & Brand.Brand<"UserId">
export const UserId = Brand.nominal<UserId>()

export type Email = string & Brand.Brand<"Email">
export const Email = Brand.nominal<Email>()
```

## The Interface (Domain Layer)

```typescript
// domain/user/UserRepository.ts
import { Effect, Context } from "effect"
import { User } from "./User"
import { UserId, Email } from "./values"
import { UserNotFound, PersistenceError } from "./errors"

export interface UserRepository {
  readonly findById: (id: UserId) => Effect.Effect<User, UserNotFound>
  readonly findByEmail: (email: Email) => Effect.Effect<User, UserNotFound>
  readonly save: (user: User) => Effect.Effect<void, PersistenceError>
  readonly delete: (id: UserId) => Effect.Effect<void, UserNotFound>
}

// Service key — the Context tag
export class UserRepository extends Context.Service<UserRepository>()(
  "UserRepository"
) {}
```

`Context.Service` makes `UserRepository` injectable via Effect's dependency injection. No DI framework needed.

Rules:
- **Domain types only** — no ORM types, no DB row shapes, no raw `string` IDs
- **One aggregate per repository** — owns `User` and children, never crosses aggregate boundaries
- **Returns domain errors** — `UserNotFound` not `null`; `PersistenceError` not `DrizzleError`
- **No query language leaks** — no `where`, no `select`, no SQL fragments

## The Implementation (Infrastructure Layer)

```typescript
// infrastructure/persistence/DrizzleUserRepository.ts
import { Effect, Layer } from "effect"
import { UserRepository } from "../../domain/user/UserRepository"
import { UserNotFound, PersistenceError } from "../../domain/user/errors"
import { toDomain, toRecord } from "./UserMapper"
import { db, users } from "../db/schema"
import { eq } from "drizzle-orm"

export const DrizzleUserRepositoryLive = Layer.succeed(
  UserRepository,
  UserRepository.of({
    findById: (id) =>
      Effect.tryPromise({
        try: () => db.select().from(users).where(eq(users.id, id)),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(
        Effect.flatMap((rows) =>
          rows[0]
            ? Effect.succeed(toDomain(rows[0]))
            : Effect.fail(new UserNotFound({ id }))
        )
      ),

    findByEmail: (email) =>
      Effect.tryPromise({
        try: () => db.select().from(users).where(eq(users.email, email)),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(
        Effect.flatMap((rows) =>
          rows[0]
            ? Effect.succeed(toDomain(rows[0]))
            : Effect.fail(new UserNotFound({ id: rows[0]?.id as any }))
        )
      ),

    save: (user) =>
      Effect.tryPromise({
        try: () => db.insert(users).values(toRecord(user)).onConflictDoUpdate({
          target: users.id,
          set: toRecord(user),
        }),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(Effect.asVoid),

    delete: (id) =>
      Effect.tryPromise({
        try: () => db.delete(users).where(eq(users.id, id)),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(Effect.asVoid),
  })
)
```

Rules:
- **Mapping is mandatory** — `toDomain()` converts DB rows → domain aggregates; `toRecord()` goes the other way → see [[ddd-mapper-pattern]]
- **No domain logic here** — invariants stay on the aggregate
- **`Layer.succeed`** provides the implementation; compose layers at the app boundary

## Usage in Application Service

```typescript
// application/FulfillOrderService.ts
import { Effect } from "effect"
import { UserRepository } from "../domain/user/UserRepository"

export const getUser = (id: UserId) =>
  Effect.gen(function* () {
    const repo = yield* UserRepository  // ← inject via Context
    const user = yield* repo.findById(id)
    return user
  }).pipe(
    Effect.catchTag("UserNotFound", (e) =>
      Effect.fail(new ApplicationError({ message: `User ${e.id} not found` }))
    )
  )
```

## Anti-patterns

| Anti-pattern | Why it's wrong |
|---|---|
| Generic `Repository<T>` with `findAll`, `query(sql)` | Leaks infrastructure into callers |
| Repository that calls another repository | Crosses aggregate boundaries |
| Service using two repositories in one transaction | Use Unit of Work at application service level |
| Repository with business logic in `save` | Invariants belong on the aggregate |
| Returning `null` / `Option` instead of typed error | Forces callers to do nil-checks; not domain language |

## On Transactions / Unit of Work

True DDD handles cross-aggregate consistency via **domain events**, not multi-repository transactions. When you must span two aggregates, use a **Unit of Work** at the application service level — both repositories participate in a shared `Effect.transaction` context.

## Related

- [[domain-driven-design]] — overview of all DDD building blocks
- [[ddd-aggregate-root]]
- [[ddd-domain-events]]
- [[ddd-mapper-pattern]] — full breakdown of toDomain / toRecord
- [[ports-and-adapters]]
- [[unit-of-work-pattern]]
- [[effect-context-and-layers]]

## References

- Evans, *Domain-Driven Design* — ch. 6
- Fowler, *Patterns of Enterprise Application Architecture* — Repository
- Effect source: `80-resources/effect-source/packages/effect/src/Context.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Data.ts`
