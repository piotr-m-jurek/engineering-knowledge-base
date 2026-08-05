---
title: DDD Request Flow — API to DB
type: architecture
created: 2026-08-05
tags: [ddd, architecture, effect, http, flow]
effect-source: 80-resources/effect-source
---

# DDD Request Flow — API to DB

How data flows through a DDD application using Effect HttpApi, from an incoming HTTP request down to the database and back.

---

## Overview

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────┐
│  HttpApi (Schema declaration)       │  defines shape, validates I/O
│  HttpApiBuilder.group (Handler)     │  receives decoded ctx, returns domain DTO
└───────────────┬─────────────────────┘
                │ decoded + typed payload
                ▼
┌─────────────────────────────────────┐
│  Application Service                │  orchestrates, no business logic
└───────────────┬─────────────────────┘
                │ domain types (value objects)
                ▼
┌─────────────────────────────────────┐
│  Aggregate (Domain)                 │  enforces invariants, emits events
└───────────────┬─────────────────────┘
                │ aggregate instance
                ▼
┌─────────────────────────────────────┐
│  Repository (port) → impl (adapter) │  persists aggregate via mapper
└───────────────┬─────────────────────┘
                │ SQL via Drizzle
                ▼
              Database
```

**Data shape at each boundary:**

| Boundary | Shape |
|---|---|
| HTTP in | Raw JSON / form data |
| Handler ctx | Decoded, validated (`Schema`) |
| App service in | Domain value objects (`Brand`) |
| Domain method | Typed aggregate + domain errors |
| Repository in | Aggregate root |
| Mapper | Aggregate ↔ DB record |
| DB | Flat primitives, snake_case |

---

## Step 1 — Declare the API (HttpApi)

Define the contract: routes, payload shapes, success/error schemas. This is pure declaration — no logic.

```typescript
// api/UsersApi.ts
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "effect/unstable/httpapi"
import { Schema } from "effect"

// --- Response schemas (what the client sees) ---

export class UserDto extends Schema.Class<UserDto>("UserDto")({
  id: Schema.String,
  email: Schema.String,
  name: Schema.String,
}) {}

// --- Error schemas ---

export class UserNotFoundError extends Schema.TaggedError<UserNotFoundError>()(
  "UserNotFoundError",
  { id: Schema.String },
  HttpApiSchema.status("NotFound")   // maps to HTTP 404
) {}

// --- API declaration ---

export class UsersApi extends HttpApiGroup.make("users")
  .add(
    HttpApiEndpoint.get("findById", "/:id", {
      params: { id: Schema.String },
      success: UserDto,
      error: UserNotFoundError,
    })
  )
  .add(
    HttpApiEndpoint.post("create", "/", {
      payload: Schema.Struct({
        email: Schema.String,
        name: Schema.String,
      }),
      success: UserDto,
    })
  )
  .prefix("/users") {}

export const Api = HttpApi.make("api").add(UsersApi)
```

Effect validates and decodes all incoming data against the schemas automatically — **handlers never receive untyped input**.

---

## Step 2 — Implement handlers (HttpApiBuilder)

Handlers receive fully decoded, typed `ctx`. Translate from API types → domain types, call the application service, translate back to DTOs.

```typescript
// api/UsersHandler.ts
import { HttpApiBuilder } from "effect/unstable/httpapi"
import { Effect } from "effect"
import { Api, UserDto, UserNotFoundError } from "./UsersApi"
import { CreateUserService, GetUserService } from "../application/UserServices"
import { UserId, Email } from "../domain/user/values"

export const UsersHandlerLive = HttpApiBuilder.group(
  Api,
  "users",
  (handlers) =>
    handlers
      .handle("findById", ({ params }) =>
        Effect.gen(function* () {
          const getUser = yield* GetUserService
          // params.id is already a decoded string — lift to domain type
          const user = yield* getUser.execute(UserId(params.id))
          // translate domain aggregate → DTO
          return new UserDto({ id: user.id, email: user.email, name: user.name })
        }).pipe(
          // map domain error → API error
          Effect.catchTag("UserNotFound", (e) =>
            Effect.fail(new UserNotFoundError({ id: e.id }))
          )
        )
      )
      .handle("create", ({ payload }) =>
        Effect.gen(function* () {
          const createUser = yield* CreateUserService
          const user = yield* createUser.execute({
            email: Email(payload.email),  // string → branded Email
            name: payload.name,
          })
          return new UserDto({ id: user.id, email: user.email, name: user.name })
        })
      )
)
```

**Handler responsibilities:**
- Lift primitive API types → domain value objects (`Brand`)
- Call application service
- Map domain errors → API errors
- Map aggregate → DTO for response

**Handler does NOT:**
- Contain business logic
- Know about the DB
- Validate business rules

---

## Step 3 — Application Service

Orchestrates the use case. Loads aggregates, calls domain methods, saves, dispatches events. No business logic.

```typescript
// application/UserServices.ts
import { Context, Effect, Layer } from "effect"
import { UserRepository } from "../domain/user/UserRepository"
import { EventBus } from "../infrastructure/EventBus"
import { User } from "../domain/user/User"
import { UserId, Email } from "../domain/user/values"
import { UserNotFound } from "../domain/user/errors"

// --- Get user ---

interface GetUserService {
  readonly execute: (id: UserId) => Effect.Effect<User, UserNotFound>
}
class GetUserService extends Context.Service<GetUserService>()("GetUserService") {}

const GetUserServiceLive = Layer.effect(
  GetUserService,
  Effect.gen(function* () {
    const repo = yield* UserRepository
    return GetUserService.of({
      execute: (id) => repo.findById(id),
    })
  })
)

// --- Create user ---

interface CreateUserInput {
  readonly email: Email
  readonly name: string
}

interface CreateUserService {
  readonly execute: (input: CreateUserInput) => Effect.Effect<User, never>
}
class CreateUserService extends Context.Service<CreateUserService>()("CreateUserService") {}

const CreateUserServiceLive = Layer.effect(
  CreateUserService,
  Effect.gen(function* () {
    const repo = yield* UserRepository
    const bus = yield* EventBus
    return CreateUserService.of({
      execute: (input) =>
        Effect.gen(function* () {
          // 1. create aggregate (enforces invariants)
          const user = yield* User.create(input)
          // 2. persist
          yield* repo.save(user)
          // 3. collect + publish events
          const events = user.pullEvents()
          yield* Effect.forEach(events, bus.publish, { discard: true })
          return user
        }),
    })
  })
)
```

---

## Step 4 — Domain (Aggregate)

Pure domain logic — no Effect requirements in `R`, no infrastructure awareness.

```typescript
// domain/user/User.ts
import { Data, Effect, Brand } from "effect"
import { UserId, Email } from "./values"
import { UserCreated } from "./events"
import { InvalidEmail } from "./errors"

export class User {
  private _events: UserCreated[] = []

  constructor(
    readonly id: UserId,
    readonly email: Email,
    readonly name: string,
  ) {}

  // Factory — validates invariants, emits event
  static create(input: {
    email: Email
    name: string
  }): Effect.Effect<User, InvalidEmail> {
    return Effect.gen(function* () {
      // invariant: name can't be empty
      if (input.name.trim().length === 0)
        yield* Effect.fail(new InvalidEmail({ message: "Name cannot be empty" }))

      const user = new User(
        UserId(crypto.randomUUID()),
        input.email,
        input.name,
      )
      user._events.push(new UserCreated({ userId: user.id, email: user.email }))
      return user
    })
  }

  pullEvents() {
    const events = [...this._events]
    this._events = []
    return events
  }
}
```

---

## Step 5 — Repository + Mapper

Repository loads/saves the aggregate. Mapper translates between DB rows and domain types.

→ Full detail in [[ddd-repository-pattern]] and [[ddd-mapper-pattern]]

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
    save: (user) =>
      Effect.tryPromise({
        try: () => db.insert(users).values(toRecord(user)).onConflictDoUpdate({
          target: users.id,
          set: toRecord(user),
        }),
        catch: (e) => new PersistenceError({ cause: e }),
      }).pipe(Effect.asVoid),
  })
)
```

---

## Step 6 — Wire everything with Layers

All the pieces above are connected at the app boundary via `Layer` composition. Nothing is wired inside the business logic.

```typescript
// main.ts
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { HttpRouter } from "effect/unstable/http"
import { HttpApiBuilder } from "effect/unstable/httpapi"
import { Layer } from "effect"
import { Api } from "./api/UsersApi"
import { UsersHandlerLive } from "./api/UsersHandler"
import { GetUserServiceLive, CreateUserServiceLive } from "./application/UserServices"
import { DrizzleUserRepositoryLive } from "./infrastructure/persistence/DrizzleUserRepository"
import { OutboxEventBusLive } from "./infrastructure/EventBus"

const AppLive = HttpRouter.serve(
  HttpApiBuilder.layer(Api).pipe(
    Layer.provide(UsersHandlerLive)
  )
).pipe(
  Layer.provide(UsersHandlerLive),
  Layer.provide(GetUserServiceLive),
  Layer.provide(CreateUserServiceLive),
  Layer.provide(DrizzleUserRepositoryLive),
  Layer.provide(OutboxEventBusLive),
  Layer.provideMerge(NodeHttpServer.layer({ port: 3000 }))
)

NodeRuntime.runMain(Layer.launch(AppLive))
```

---

## Data shapes at each boundary (summary)

```
POST /users  { "email": "a@b.com", "name": "Alice" }
      │
      │  raw JSON
      ▼
HttpApiEndpoint schema decodes + validates
      │
      │  { email: string, name: string }  (typed, validated)
      ▼
Handler — lifts to domain types
      │
      │  { email: Email, name: string }  (branded)
      ▼
Application service → User.create()
      │
      │  User instance + UserCreated event
      ▼
repo.save(user) → toRecord(user)
      │
      │  { id: string, email: string, name: string, created_at: Date }
      ▼
Drizzle INSERT → PostgreSQL
      │
      ▼  (response path — reverse)
toDomain(row) → User → UserDto → JSON response
```

---

## Key rules per layer

| Layer | Allowed | NOT allowed |
|---|---|---|
| Handler | Decode, lift to domain types, map errors/responses | Business logic, DB access |
| App service | Orchestrate: load → call domain → save → publish | Business rules, SQL |
| Domain | Invariants, state transitions, emit events | I/O of any kind |
| Repository | Load/save aggregates, map errors | Business logic, cross-aggregate queries |
| Mapper | Structural translation only | Validation, logic, conditionals |

---

## Related

- [[domain-driven-design]]
- [[ddd-repository-pattern]]
- [[ddd-mapper-pattern]]
- [[ddd-aggregate-root]]
- [[ddd-domain-events]]
- [[unit-of-work-pattern]]
- [[ports-and-adapters]]

## References

- Effect source: `80-resources/effect-source/packages/effect/src/unstable/httpapi/HttpApiBuilder.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/unstable/httpapi/HttpApiEndpoint.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/unstable/httpapi/HttpApi.ts`
