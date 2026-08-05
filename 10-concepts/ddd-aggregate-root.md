---
title: DDD Aggregate Root
type: concept
created: 2026-08-05
tags: [ddd, domain-model, effect]
effect-source: 80-resources/effect-source
---

# DDD Aggregate Root

## What

A cluster of domain objects treated as a single unit. The **aggregate root** is the only entry point from outside — all operations go through it, enforcing invariants for the whole cluster.

## Rules

- External objects hold references only to the root, never to internal members
- Transactions don't cross aggregate boundaries
- The root enforces all invariants for the cluster
- Load and save whole aggregates (never partial)

## Example with Effect

```typescript
import { Data, Effect, Brand } from "effect"

// --- Value objects & IDs ---

type OrderId = string & Brand.Brand<"OrderId">
const OrderId = Brand.nominal<OrderId>()

type ProductId = string & Brand.Brand<"ProductId">
const ProductId = Brand.nominal<ProductId>()

class Quantity extends Data.Class<{ readonly value: number }> {}

class Money extends Data.Class<{
  readonly amount: number
  readonly currency: "USD" | "EUR"
}> {}

// --- Errors ---

class OrderAlreadyFulfilled extends Data.TaggedError("OrderAlreadyFulfilled")<{
  readonly orderId: OrderId
}> {}

class InvalidQuantity extends Data.TaggedError("InvalidQuantity")<{
  readonly value: number
}> {}

// --- Internal child (never referenced directly from outside) ---

class OrderLine extends Data.Class<{
  readonly product: ProductId
  readonly qty: Quantity
  readonly unitPrice: Money
}> {}

// --- Domain events ---

class OrderLineAdded extends Data.TaggedClass("OrderLineAdded")<{
  readonly orderId: OrderId
  readonly product: ProductId
  readonly qty: Quantity
}> {}

class OrderFulfilled extends Data.TaggedClass("OrderFulfilled")<{
  readonly orderId: OrderId
}> {}

// --- Aggregate root ---

type OrderStatus = "draft" | "fulfilled" | "cancelled"
type DomainEvent = OrderLineAdded | OrderFulfilled

class Order {
  private _events: DomainEvent[] = []

  constructor(
    readonly id: OrderId,
    private _lines: OrderLine[],
    private _status: OrderStatus
  ) {}

  get status() { return this._status }
  get lines() { return [...this._lines] } // defensive copy

  addLine(
    product: ProductId,
    qty: Quantity,
    unitPrice: Money
  ): Effect.Effect<void, OrderAlreadyFulfilled | InvalidQuantity> {
    return Effect.gen(function* (this: Order) {
      if (this._status === "fulfilled")
        yield* Effect.fail(new OrderAlreadyFulfilled({ orderId: this.id }))
      if (qty.value <= 0)
        yield* Effect.fail(new InvalidQuantity({ value: qty.value }))
      this._lines.push(new OrderLine({ product, qty, unitPrice }))
      this._events.push(new OrderLineAdded({ orderId: this.id, product, qty }))
    }.bind(this))
  }

  fulfill(): Effect.Effect<void, OrderAlreadyFulfilled> {
    return Effect.gen(function* (this: Order) {
      if (this._status === "fulfilled")
        yield* Effect.fail(new OrderAlreadyFulfilled({ orderId: this.id }))
      this._status = "fulfilled"
      this._events.push(new OrderFulfilled({ orderId: this.id }))
    }.bind(this))
  }

  // Application service calls this to get events after saving
  pullEvents(): DomainEvent[] {
    const events = [...this._events]
    this._events = []
    return events
  }
}
```

## Sizing aggregates

Keep them **small**. An aggregate is a consistency boundary, not a grouping of convenience. If two things don't need to change atomically, they probably belong in different aggregates.

> Common mistake: `User` aggregate containing `Orders[]`. Orders have their own lifecycle — they should be a separate aggregate referencing `UserId`.

Ask: *"Do these two things need to change in the same transaction?"* If no — different aggregates.

## Testing without infrastructure

Because aggregates are pure domain objects with no Effect requirements in `R`, they're trivial to test:

```typescript
import { Effect } from "effect"
import { expect, test } from "vitest"

test("cannot add line to fulfilled order", () =>
  Effect.gen(function* () {
    const order = new Order(OrderId("1"), [], "draft")
    yield* order.fulfill()
    const result = yield* order.addLine(
      ProductId("p1"),
      new Quantity({ value: 1 }),
      new Money({ amount: 10, currency: "USD" })
    ).pipe(Effect.either)

    expect(result._tag).toBe("Left")
    if (result._tag === "Left")
      expect(result.left._tag).toBe("OrderAlreadyFulfilled")
  }).pipe(Effect.runPromise)
)
```

## Related

- [[domain-driven-design]]
- [[ddd-repository-pattern]] — one repository per aggregate root
- [[ddd-domain-events]] — aggregates emit events, never call other aggregates

## References

- Fowler, [DDD_Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- Evans, *DDD* ch. 6
- Effect source: `80-resources/effect-source/packages/effect/src/Data.ts`
- Effect source: `80-resources/effect-source/packages/effect/src/Brand.ts`
