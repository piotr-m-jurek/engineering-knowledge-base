---
title: Infrastructure Adapter — DDD meaning
type: concept
created: 2026-08-11
tags: [ddd, architecture, ports-adapters, effect, coupling, infrastructure]
---

# Infrastructure Adapter — DDD meaning

> "This is **fine** when `TodoRepo` is an infrastructure adapter in the same deployment (a DB wrapper)."
> — [[service-coupling]]

That sentence draws a line between two kinds of coupling. Understanding it requires knowing how DDD splits the codebase into layers and what "infrastructure" means in that split.

---

## The DDD layer map

DDD organises code into four layers:

```
┌──────────────────────────────────────────────────────────────┐
│  Presentation / Driving Layer                                │
│  HTTP handlers, CLI, queues consuming input                  │
├──────────────────────────────────────────────────────────────┤
│  Application Layer                                           │
│  Use cases — orchestrates domain, no business rules          │
├──────────────────────────────────────────────────────────────┤
│  Domain Layer                                                │
│  Aggregates, domain services, ports (interfaces)             │
│  ← the core; no dependencies on anything below               │
├──────────────────────────────────────────────────────────────┤
│  Infrastructure Layer                                        │
│  Adapters: DB drivers, HTTP clients, file system,            │
│  message brokers, email senders                              │
└──────────────────────────────────────────────────────────────┘
```

The dependency rule: **arrows point inward only**. Infrastructure depends on domain. Domain depends on nothing outside itself.

---

## What "infrastructure adapter" means

An **infrastructure adapter** is a class/module in the infrastructure layer that implements a **port** (interface) defined in the domain layer. It translates between the domain's language and an external system.

Canonical example — a repository:

```
Domain layer:
  UserRepository  ← interface (port) — domain defines what it needs

Infrastructure layer:
  DrizzleUserRepository  ← class (adapter) — infrastructure implements it using Drizzle + Postgres
```

From [[ddd-repository-pattern]]:

```typescript
// domain/user/UserRepository.ts  ← PORT (defined in domain)
export class UserRepository extends Context.Service<UserRepository>()(
  "UserRepository"
) {}

// infrastructure/persistence/DrizzleUserRepository.ts  ← ADAPTER (implements it)
export const DrizzleUserRepositoryLive = Layer.succeed(
  UserRepository,
  UserRepository.of({
    findById: (id) => Effect.tryPromise({ try: () => db.select()... })
  })
)
```

The domain only knows the port interface. It never imports Drizzle. The adapter is the thing that does.

---

## Why this coupling is "fine"

In [[service-coupling]], `TodoService` acquiring `TodoRepo` via `yield*` is used as an example of coupling that is acceptable:

```typescript
// from: 80-resources/effect-source/ai-docs/src/09_testing/20_layer-tests.ts
// TodoService yields TodoRepo — this is a direct dependency
const repo = yield* TodoRepo
```

This looks like coupling. It is coupling. But it's **acceptable** because of what `TodoRepo` *is* in the DDD layer model:

1. **It is not an independent service** — it has no lifecycle of its own, no team of its own, no deployment of its own. It is a wrapper around Postgres that lives inside the same process.

2. **It cannot "be down" independently** — if the DB is down, the whole application is down. The coupling doesn't add a new failure mode; the failure mode was already there.

3. **It changes only when persistence changes** — renaming a method on `TodoRepo` is an internal refactor within one deployment unit, not a cross-team contract negotiation.

4. **It is the correct direction of dependency** — `TodoService` (application layer) depending on `TodoRepo` (infrastructure adapter fulfilling a domain port) is exactly what the layer model prescribes.

Compare this to the problematic case:

```
✓ Fine:    OrderService → OrderRepository (infrastructure adapter, same process, same deploy)
✗ Problem: OrderService → InventoryService (independent service, different process, different deploy)
```

The second case is where the failure modes diverge: `InventoryService` can be redeployed, have its API changed by another team, or be temporarily down while `OrderService` runs fine. The dependency crosses a **deployment boundary**, which is what makes it harmful coupling.

---

## The key question

> "Is this dependency crossing a deployment boundary?"

| Dependency | Deployment boundary crossed? | Coupling type |
|---|---|---|
| `TodoService` → `TodoRepo` (Drizzle) | No — same process | Structural only, acceptable |
| `TodoService` → `UserService` (separate microservice) | Yes | Temporal + structural, problematic |
| `TodoService` → `TodoRepo` (remote gRPC repo service) | Yes | Same as microservice |
| `OrderService` → `EmailAdapter` (wraps SendGrid) | No — same process | Structural only, acceptable |

An infrastructure adapter is by definition **not** across a deployment boundary — it is compiled into and runs within the same application binary.

---

## The "same deployment" qualifier

The sentence says "in the same deployment." This matters because the same pattern (`yield* SomeRepo`) can be either fine or broken depending on where `SomeRepo` lives:

- **In-process DB adapter** (Drizzle, raw `pg`, etc.) → fine. It's infrastructure.
- **Remote repository service** (a microservice that manages data and exposes an API) → problematic. It's an independent service pretending to be a repo.

"Infrastructure adapter" is not just about the word — it's about topology. The test: *can this thing fail independently of my service?* If yes, it needs decoupling. If no, the `yield*` is fine.

---

## Summary

"Infrastructure adapter" in DDD = a module that:
- Implements a port (interface) defined in the domain layer
- Lives in the infrastructure layer of the same deployment
- Talks to one external system (DB, cache, email, storage) on behalf of the domain
- Has no business logic — only translation and I/O

When `TodoService` depends on `TodoRepo`, it depends on one of these. That's not problematic service coupling — it's the intended dependency direction in the DDD layer model. Problematic coupling is when the dependency crosses into another autonomous service with its own deployment lifecycle.

---

## Related

- [[service-coupling]] — the note this concept clarifies; where to use EDA vs API design instead
- [[ports-and-adapters]] — the full hexagonal architecture pattern; ports vs adapters defined
- [[ddd-repository-pattern]] — canonical infrastructure adapter example with Effect v4
- [[domain-driven-design]] — DDD layer model overview
- [[ddd-bounded-context]] — deployment boundary = bounded context boundary in strategic DDD
