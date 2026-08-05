---
title: DDD Aggregate Root
type: concept
created: 2026-08-05
tags: [ddd, domain-model]
---

# DDD Aggregate Root

## What

A cluster of domain objects treated as a single unit. The **aggregate root** is the only entry point from outside — all operations go through it, enforcing invariants for the whole cluster.

## Rules

- External objects hold references only to the root, never to internal members
- Transactions don't cross aggregate boundaries
- The root enforces all invariants for the cluster
- Load and save whole aggregates (never partial)

## Example

```typescript
class Order {  // aggregate root
  private lines: OrderLine[] = []

  addLine(product: ProductId, qty: Quantity): void {
    // invariant: can't add to a fulfilled order
    if (this.status === OrderStatus.Fulfilled)
      throw new OrderAlreadyFulfilled(this.id)
    this.lines.push(new OrderLine(product, qty))
  }
}

class OrderLine {  // internal — never referenced directly from outside
  constructor(readonly product: ProductId, readonly qty: Quantity) {}
}
```

## Sizing aggregates

Keep them small. An aggregate is a **consistency boundary**, not a convenience grouping. If two things don't need to change atomically, they probably belong in different aggregates.

> Common mistake: `User` aggregate containing `Orders[]`. Orders have their own lifecycle — they should be a separate aggregate referencing `UserId`.

## Related

- [[domain-driven-design]]
- [[ddd-repository-pattern]] — one repository per aggregate root
- [[ddd-domain-events]] — cross-aggregate consistency without transactions

## References

- Fowler, [DDD_Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- Evans, *DDD* ch. 6
