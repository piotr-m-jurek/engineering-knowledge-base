---
title: Architecture Decision Records (ADRs)
type: concept
created: 2026-08-11
tags: [architecture, decision, documentation, process, team]
---

# Architecture Decision Records (ADRs)

## What

A short document that captures **one significant architectural decision**: the context that forced the choice, the decision itself, and the consequences — both positive and negative.

Coined by Michael Nygard (2011). The format is intentionally lightweight — one page, not a design doc. The goal is a log you can read years later and understand *why* the codebase looks the way it does.

---

## The problem it solves

Code tells you *what* the system does. Comments sometimes tell you *how*. Nothing tells you *why*.

> Why is the event store append-only and we never delete? Why do we use Kafka instead of a simple job table? Why does the domain layer have no direct dependency on the ORM?

Six months later, the person who made that call has left or forgotten. A new engineer "cleans up" the seemingly arbitrary constraint and breaks something subtle. Or worse — they don't clean it up because they're afraid, and now you have undocumented load-bearing folklore.

ADRs make decisions explicit, durable, and locatable.

---

## The format (as used in this vault)

From `90-templates/adr.md`:

```markdown
---
title: ADR title
type: adr
status: proposed | accepted | deprecated | superseded
---

## Context
What situation forced this decision? What constraints exist?
What was the pain point or the requirement?

## Decision
What did we decide? State it as a clear, active sentence.
"We will use X." Not "X was considered."

## Consequences

### Positive
### Negative
### Neutral

## Alternatives considered
| Option | Why rejected |

## Related
```

**Status lifecycle:**
- `proposed` — written, not yet agreed
- `accepted` — agreed, in effect
- `deprecated` — no longer applies, but kept for history
- `superseded` — replaced by a newer ADR (link to it)

ADRs are **never deleted** — even wrong decisions are kept, marked `superseded`, so you can trace the reasoning chain.

---

## What counts as "significant"

Not every choice needs an ADR. A decision is worth recording when:

- It is **hard to reverse** — switching databases, choosing an event model, splitting a monolith
- It **rules out alternatives** that look equally valid — why Kafka and not RabbitMQ?
- It **constrains future decisions** — "domain layer must not import infrastructure" shapes all future code
- It will **surprise a new engineer** — anything that looks wrong until you understand the context
- It involves a **deliberate trade-off** you want to remember — "we accepted eventual consistency here because..."

Routine choices (which library to format dates, which linter rules) don't need ADRs.

---

## Where they live in this vault

All ADRs go in `60-decisions/`. The [[_mocs/decisions]] MOC has a Dataview table of all ADRs sorted by status and date.

Naming convention: `NNNN-short-title.md` — e.g. `0001-use-event-sourcing-for-budgets.md`. The number gives a chronological order and a stable reference ("see ADR-0001").

---

## The relationship to architecture notes

ADRs in `60-decisions/` are **project- or team-specific** decisions with a date and a status. Concept notes in `10-concepts/` are **timeless explanations** of what a pattern is.

```
10-concepts/event-sourcing.md       ← what event sourcing is and when to use it
60-decisions/0001-use-event-sourcing-for-budgets.md  ← we decided to use it here, on this date, for these reasons
```

The ADR links to the concept. The concept does not link to specific ADRs (it doesn't know about them).

---

## Practical tips

**Write it before the decision, not after.** An ADR written after the fact is archaeology. Written before, it forces you to articulate the trade-offs while you still have options.

**One decision per ADR.** If you find yourself writing "and also we decided...", that's a second ADR.

**Be honest about negatives.** An ADR that lists no negative consequences is probably incomplete. Every significant architectural choice has costs — record them so future engineers know what they're living with.

**Link forward when superseding.** If ADR-0003 replaces ADR-0001, update ADR-0001 to say `superseded by [[0003-...]]`. The chain stays navigable.

**Short is better.** One page. If it's longer, you're writing a design doc, not an ADR.

---

## Example skeleton

> **ADR-0001: Use event sourcing for the budget aggregate**
>
> **Status:** accepted — 2026-07-01
>
> **Context:** We need full audit history of all budget changes for regulatory reasons. We also want time-travel queries ("what was the budget on July 15?"). A mutable `budgets` table with an audit trigger loses intermediate states and requires reconstructing history from diffs.
>
> **Decision:** We will store budget state as an append-only sequence of domain events in `budget_events`. Current state is derived by replaying events. Snapshots are taken every 50 events for read performance.
>
> **Consequences:**
> - ✓ Full audit log for free — history is the primary data
> - ✓ Time-travel queries trivial — replay to any point
> - ✗ No direct `SELECT current_state FROM budgets` — projections required
> - ✗ Event schema must be forwards-compatible forever, or upcasters needed
>
> **Alternatives considered:**
> | Option | Why rejected |
> |--------|-------------|
> | Mutable table + audit trigger | Loses intermediate states; audit is an afterthought |
> | Temporal tables | DB-vendor specific; doesn't capture domain semantics |
>
> **Related:** [[event-sourcing]], [[ddd-domain-events]]

---

## Related

- [[_mocs/decisions]] — all ADRs in this vault
- [[event-sourcing]] — a common subject of ADRs; costly to reverse
- [[durable-execution]] — another costly-to-reverse commitment worth an ADR
- [[architecture-overkill-guide]] — helps identify whether the decision is even necessary
- [[domain-driven-design]] — bounded context boundaries are classic ADR territory
- [[ports-and-adapters]] — "all infrastructure behind interfaces" is often ADR-001 in a DDD project

## References

- Nygard, [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) (2011) — the original post
- Thoughtworks Technology Radar — ADRs have been on "adopt" since 2020
- `90-templates/adr.md` — the template used in this vault
