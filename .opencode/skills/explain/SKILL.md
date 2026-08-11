---
name: explain
description: >
  Junior-to-principal explanation style for this knowledge vault.
  Produces a layered walkthrough of a concept at four levels of understanding.
  Use when user writes /explain <topic>, or asks to "explain X following the junior to senior style",
  or asks to "explain X for the vault".
---

# /explain skill

Produce a single note-quality explanation of the topic walking through four levels:

## Levels

1. **Junior** — intuition and analogy. One simple code snippet max. No jargon without definition.
2. **Mid-level** — real implementation. Use Effect v4 (`Context.Service`, `Layer.effect`, `Effect.fn`, `Effect.gen`) when the topic involves decoupling, services, or dependency injection. Plain TypeScript for pure domain types or algorithms.
3. **Senior** — trade-offs, failure modes, scaling implications, eventual consistency concerns. Compare alternatives.
4. **Principal** — when to use vs when not to. Boundary decisions. Team/org implications. Cost of adoption.

## Effect v4 rules in code examples

- Services: `class Foo extends Context.Service<Foo>()("namespace/Foo", { ... })()`
- Layers: `static readonly layer = Layer.effect(Foo, Effect.gen(...))`
- Named functions: `Effect.fn("Foo.method")(function* (...) { ... })`
- Yield errors directly: `yield* new MyError({...})` — no `Effect.fail` wrapper
- No `Context.Tag`, no `Layer.scoped`, no `@effect/platform` imports (removed in v4)
- Imports: `import { Context, Effect, Layer } from "effect"`

## Output format

- Use markdown headers matching the level names above
- End with a **Related** section linking to other vault notes via `[[wikilink]]`
- End with a **References** section (books, canonical talks, authoritative blog posts)

## When saving to vault

- File goes in `10-concepts/` (timeless concept) or `30-patterns/` (reusable design pattern)
- Add frontmatter: `title`, `type`, `created: YYYY-MM-DD`, `tags`
- Link generously to existing notes
- Do not put version-specific implementation code in `10-concepts/` — it goes in `70-projects/`
