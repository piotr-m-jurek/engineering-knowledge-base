---
title: DDD Bounded Context
type: concept
created: 2026-08-05
tags: [ddd, strategic-design]
---

# DDD Bounded Context

## What

A boundary within which a domain model is internally consistent and the ubiquitous language is unambiguous. The primary tool of DDD's **strategic design**.

Large systems can't have one unified model — the same word means different things in different parts of the business. Bounded contexts make those boundaries explicit.

## Example

`Customer` is a polyseme:

| Context | What "Customer" means |
|---|---|
| Billing | Entity with payment method, billing address, invoices |
| Shipping | Entity with delivery address, preferred carrier |
| Support | Entity with ticket history, SLA tier |

Each context has its own `Customer` model. That's correct, not a mistake.

## Context Map

Diagram showing all bounded contexts and their relationships:

- **Shared Kernel** — two contexts share a small common model, changes require both teams
- **Customer/Supplier** — upstream context publishes, downstream consumes
- **Conformist** — downstream adopts upstream model as-is
- **Anti-Corruption Layer (ACL)** — downstream translates upstream model to its own; protects domain purity
- **Open Host Service** — upstream publishes a formal API for multiple consumers
- **Separate Ways** — no integration; contexts evolve independently

## Relation to microservices

A bounded context is a good candidate for a microservice boundary — but not a rule. One service can contain multiple bounded contexts, or a context can span services. Conway's Law applies: boundaries often follow team boundaries.

## Related

- [[domain-driven-design]]
- [[ddd-domain-events]] — primary integration mechanism between contexts
- [[ports-and-adapters]]

## References

- Fowler, [BoundedContext](https://martinfowler.com/bliki/BoundedContext.html)
- Evans, *DDD* Part IV — Strategic Design
- Vernon, *IDDD* ch. 2–3
