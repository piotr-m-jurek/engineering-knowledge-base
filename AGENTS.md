# Agent Instructions — Programming Knowledge Vault

## Session start checklist

Run at the start of every session before doing any work:

```bash
git -C 80-resources/effect-source pull --ff-only origin main
```

If it fails (conflicts, diverged), report it and continue — do not force. If the pull succeeds, note the new HEAD so references to source files stay accurate.

---

## Vault structure

```
_mocs/          Maps of Content — entry points, not notes
00-inbox/       Unprocessed captures
10-concepts/    Timeless principles: CS fundamentals, mental models (e.g. DDD building blocks)
20-languages/   Language-specific notes: TypeScript, Kotlin, Rust, etc.
30-patterns/    Design and architectural patterns (e.g. repository, ports-and-adapters)
40-tools/       CLIs, editors, build tools, testing frameworks
50-architectures/ System design, distributed systems, frontend architecture
60-decisions/   ADRs — Architecture Decision Records
70-projects/    Concrete project blueprints with implementation detail
80-resources/   External source trees checked in as git submodules
90-templates/   Obsidian Templater templates — do not edit notes here
```

### Where things go

| Content type | Folder |
|---|---|
| Timeless concept (DDD, CQRS, recursion) | `10-concepts/` |
| Language idiom or stdlib knowledge | `20-languages/` |
| Reusable pattern with code | `30-patterns/` |
| Tool reference or config | `40-tools/` |
| System / architecture design | `50-architectures/` |
| Technology decision with rationale | `60-decisions/` |
| Project blueprint with specific API code | `70-projects/` |
| Raw capture needing processing | `00-inbox/` |

Concepts survive library version changes. Projects are version-pinned and reference specific source.

---

## Note conventions

Frontmatter required on every note:

```yaml
---
title: Note Title
type: concept | pattern | language | tool | architecture | decision | project | moc
created: YYYY-MM-DD
tags: []
---
```

Use `[[wikilink]]` for internal links. Link generously — notes should be densely connected.

Code blocks must specify language: ` ```typescript `, ` ```bash `, etc.

---

## Effect source resource

Path: `80-resources/effect-source/`
Remote: `https://github.com/Effect-TS/effect.git`
Current version: **Effect v4**

### Key entry points in the source

| What to look up | Where |
|---|---|
| Full API overview | `80-resources/effect-source/LLMS.md` |
| v3 → v4 migration | `80-resources/effect-source/MIGRATION.md` |
| Runnable examples | `80-resources/effect-source/ai-docs/` |
| Cookbooks | `80-resources/effect-source/cookbooks/` |
| Core module source | `80-resources/effect-source/packages/effect/src/` |
| HTTP (unstable) | `80-resources/effect-source/packages/effect/src/unstable/http/` |

### Critical v4 API facts

- `Context.Tag` **removed** → use `Context.Service`
- `Effect.Tag` **removed** → use `Context.Service`
- `Layer.scoped` **removed** → `Layer.effect` handles Scope automatically
- `@effect/platform` HTTP **merged into** `effect` → import from `effect/unstable/http`
- `Either` **renamed** → `Result`
- Service proxy accessor **removed** → `yield* Service` or `Service.use(fn)`
- Service `.Default` layer convention → `.layer`
- Named effectful functions: use `Effect.fn("name")(function*() {...})` — adds spans and stack traces
- Domain errors yieldable directly: `yield* new MyError({...})` inside `Effect.gen` — no `Effect.fail` wrapper needed

### Service definition pattern (v4)

```typescript
import { Context, Effect, Layer } from "effect"

class MyService extends Context.Service<MyService, {
  doThing(x: string): Effect.Effect<Result, MyError>
}>()("myapp/MyService") {
  static readonly layer = Layer.effect(
    MyService,
    Effect.gen(function* () {
      return MyService.of({
        doThing: Effect.fn("MyService.doThing")(function* (x) {
          return yield* Effect.succeed("done")
        }),
      })
    })
  )
}
```

### When writing Effect code in notes

1. Check `LLMS.md` first for the canonical pattern
2. Check `ai-docs/` for a working example
3. Fall back to package source only for edge cases
4. Always verify import paths — v4 moves many things

---

## DDD notes cluster

The vault has a DDD cluster. When adding DDD-related notes, link to:

- `[[domain-driven-design]]` — main concept hub
- `[[ddd-aggregate-root]]`
- `[[ddd-bounded-context]]`
- `[[ddd-domain-events]]`
- `[[ddd-repository-pattern]]` (in `30-patterns/`)
- `[[ports-and-adapters]]` (in `30-patterns/`)
- `[[70-projects/ddd-finance-app]]` — reference implementation with v4 code

---

## Note writing style

When writing concept notes, use the `/explain` style — a single note that walks through the concept at increasing levels of understanding:

1. **Junior** — intuition, simple analogy, minimal code
2. **Mid-level** — real implementation with Effect v4 examples showing decoupling
3. **Senior** — trade-offs, scaling implications, eventual consistency, system thinking
4. **Principal** — when to use vs not, boundary decisions, team/org implications

Code examples in notes use **Effect v4** as the default language when illustrating decoupling, service design, or dependency injection patterns. Plain TypeScript is fine for pure domain types or algorithms that don't involve effects.

Trigger: user writes `/explain <topic>` or asks to "explain X following the junior to senior style".

---

## Do not

- Edit files in `90-templates/` — those are Obsidian Templater templates
- Edit files in `.obsidian/` — Obsidian config
- Commit to `80-resources/effect-source/` — it's a submodule, pull only
- Put version-specific implementation code in `10-concepts/` — it goes in `70-projects/`
- Use `Context.Tag`, `Layer.scoped`, or `@effect/platform` imports — all removed in v4
