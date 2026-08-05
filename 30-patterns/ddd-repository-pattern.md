---
title: DDD Repository Pattern
type: pattern
created: 2026-08-05
tags: [ddd, architecture, persistence, ports-and-adapters]
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

## The Interface (Domain Layer)

```typescript
// domain/user/UserRepository.ts
export interface UserRepository {
  findById(id: UserId): Effect.Effect<User, UserNotFound>
  findByEmail(email: Email): Effect.Effect<User, UserNotFound>
  save(user: User): Effect.Effect<void, PersistenceError>
  delete(id: UserId): Effect.Effect<void, UserNotFound>
}
```

Rules:
- **Domain types only** — no ORM types, no DB row shapes, no `string` IDs where the domain uses value objects
- **One aggregate per repository** — `UserRepository` owns `User` and its children; never crosses aggregate boundaries
- **Returns domain errors** — `UserNotFound` not `null`; `PersistenceError` not `DrizzleError`
- **No query language leaks** — no `where`, no `select`, no SQL fragments

## The Implementation (Infrastructure Layer)

```typescript
// infrastructure/persistence/DrizzleUserRepository.ts
export class DrizzleUserRepository implements UserRepository {
  findById(id: UserId) {
    return Effect.tryPromise({
      try: () => db.select().from(users).where(eq(users.id, id.value)),
      catch: (e) => new PersistenceError(e)
    }).pipe(
      Effect.flatMap(rows =>
        rows[0] ? Effect.succeed(toDomain(rows[0])) : Effect.fail(new UserNotFound(id))
      )
    )
  }
}
```

Rules:
- **Mapping is mandatory** — `toDomain()` converts DB rows → domain aggregates; `toRecord()` converts aggregates → DB rows
- **No domain logic here** — invariants, validation, business rules stay in the aggregate
- **Implements the interface exactly** — no extra methods the domain didn't define

## Anti-patterns

| Anti-pattern | Why it's wrong |
|---|---|
| Generic `Repository<T>` with `findAll`, `query(sql)` | Leaks infrastructure concepts into callers |
| Repository that calls another repository | Crosses aggregate boundaries |
| Service that uses two repositories in a transaction | Transaction coordination belongs in an application service with a unit of work |
| Repository with business logic (`save` validates invariants) | Invariants belong on the aggregate |
| Returning `null` instead of a typed error | Forces nil-checks on callers; not domain language |

## On Transactions / Unit of Work

True DDD handles cross-aggregate consistency via **domain events**, not multi-repository transactions. If you must span two aggregates in one transaction, use a **Unit of Work** at the application service level — repositories participate in a shared transaction context, neither orchestrates the other.

## Summary

- Interface in domain, implementation in infrastructure
- Speaks domain language (value objects, domain errors)
- One aggregate root per repository
- No ORM/SQL leaks across the boundary
- Mapping layer is the bridge between worlds

## Related

- [[domain-driven-design]] — overview of all DDD building blocks
- [[ddd-aggregate-root]]
- [[ddd-domain-events]]
- [[ports-and-adapters]]
- [[unit-of-work-pattern]]

## References

- Evans, *Domain-Driven Design* — ch. 6
- Fowler, *Patterns of Enterprise Application Architecture* — Repository
