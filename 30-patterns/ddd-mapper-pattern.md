---
title: DDD Mapper Pattern (toDomain / toRecord)
type: pattern
created: 2026-08-05
tags: [ddd, persistence, mapping, effect]
effect-source: 80-resources/effect-source
---

# DDD Mapper Pattern (`toDomain` / `toRecord`)

## What

A pure translation layer between the **infrastructure representation** (DB row — flat, all primitives) and the **domain representation** (typed aggregate — branded IDs, value objects, class instances).

Lives in the infrastructure layer alongside the repository implementation. Contains **zero business logic** — only structural translation.

```
DB row (UserRecord)   ──toDomain──▶   User (aggregate)
User (aggregate)      ──toRecord──▶   DB row (UserRecord)
```

→ Called from [[ddd-repository-pattern]]; never called from the domain layer directly.

---

## Simple example: `User` aggregate

```typescript
// infrastructure/persistence/UserMapper.ts
import { Brand } from "effect"
import { User } from "../../domain/user/User"
import { UserId, Email } from "../../domain/user/values"

// What Drizzle gives back — flat, all primitives, snake_case
type UserRecord = {
  id: string
  email: string
  name: string
  created_at: Date
}

// DB row → domain aggregate
export const toDomain = (row: UserRecord): User =>
  new User({
    id: UserId(row.id),       // string → branded UserId
    email: Email(row.email),  // string → branded Email
    name: row.name,
  })

// Domain aggregate → DB row
export const toRecord = (user: User): UserRecord => ({
  id: user.id,                // UserId is a string at runtime — no unwrap needed
  email: user.email,
  name: user.name,
  created_at: new Date(),
})
```

`Brand.nominal` creates zero-cost branded types — at runtime `UserId` is still a plain `string`. The brand only exists in the type system, so `toRecord` can use `user.id` directly without unwrapping.

---

## Complex example: `Order` aggregate with children

Aggregates with child collections require joining rows first, then mapping together:

```typescript
// infrastructure/persistence/OrderMapper.ts
import { Data, Brand } from "effect"
import { Order } from "../../domain/order/Order"
import { OrderLine } from "../../domain/order/OrderLine"
import { OrderId, ProductId } from "../../domain/order/values"
import { Money, Quantity } from "../../domain/shared/values"

type OrderRecord = {
  id: string
  customer_id: string
  status: string           // stored as string in DB
}

type OrderLineRecord = {
  id: string
  order_id: string
  product_id: string
  quantity: number
  unit_price_amount: number
  unit_price_currency: "USD" | "EUR"
}

// Both the root row and child rows passed in together
export const toDomain = (
  row: OrderRecord,
  lines: OrderLineRecord[]
): Order =>
  new Order(
    OrderId(row.id),
    lines.map((l) =>
      new OrderLine({
        product: ProductId(l.product_id),
        qty: new Quantity({ value: l.quantity }),
        unitPrice: new Money({
          amount: l.unit_price_amount,
          currency: l.unit_price_currency,
        }),
      })
    ),
    row.status as OrderStatus,  // narrow string → union type
  )

export const toRecord = (order: Order): {
  order: OrderRecord
  lines: Omit<OrderLineRecord, "id">[]
} => ({
  order: {
    id: order.id,
    customer_id: order.customerId,
    status: order.status,
  },
  lines: order.lines.map((l) => ({
    order_id: order.id,
    product_id: l.product.toString(),
    quantity: l.qty.value,
    unit_price_amount: l.unitPrice.amount,
    unit_price_currency: l.unitPrice.currency,
  })),
})
```

The repository is responsible for the join query; the mapper only deals with the translation:

```typescript
// In DrizzleOrderRepository
findById: (id) =>
  Effect.tryPromise({
    try: async () => {
      const [orderRow] = await db.select().from(orders).where(eq(orders.id, id))
      const lineRows = await db.select().from(orderLines).where(eq(orderLines.order_id, id))
      return { orderRow, lineRows }
    },
    catch: (e) => new PersistenceError({ cause: e }),
  }).pipe(
    Effect.flatMap(({ orderRow, lineRows }) =>
      orderRow
        ? Effect.succeed(toDomain(orderRow, lineRows))  // ← mapper called here
        : Effect.fail(new OrderNotFound({ id }))
    )
  ),
```

---

## With Schema validation (optional, stricter)

If you want runtime validation at the DB boundary (e.g. you don't fully trust the DB schema), use `Schema` to parse the row:

```typescript
import { Schema, Effect } from "effect"

const UserRecordSchema = Schema.Struct({
  id: Schema.String,
  email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+$/)),
  name: Schema.String,
  created_at: Schema.DateFromSelf,
})

export const toDomain = (row: unknown): Effect.Effect<User, Schema.ParseError> =>
  Schema.decodeUnknown(UserRecordSchema)(row).pipe(
    Effect.map((validated) =>
      new User({
        id: UserId(validated.id),
        email: Email(validated.email),
        name: validated.name,
      })
    )
  )
```

Only worth the overhead if the DB is shared across multiple writers or the schema is unreliable.

---

## Rules

- **Dumb by design** — no business logic, no invariant checks, no conditionals beyond structural translation
- **Infrastructure layer only** — domain layer never imports mapper
- **One mapper per aggregate** — `UserMapper.ts`, `OrderMapper.ts`, etc.
- **Flat ↔ rich** — DB rows are flat primitives; domain aggregates are rich typed objects. The mapper is the only place this translation happens.

---

## Related

- [[ddd-repository-pattern]] — mapper is called inside the repository implementation
- [[ddd-aggregate-root]] — what `toDomain` reconstructs
- [[domain-driven-design]] — where this fits in the layered architecture

## References

- Fowler, *Patterns of Enterprise Application Architecture* — Data Mapper pattern
- Effect source: `80-resources/effect-source/packages/effect/src/Brand.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Schema.ts`
