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
2. **Mid-level** — real implementation. Use Effect v4 when the topic involves decoupling, services, or dependency injection. Plain TypeScript for pure domain types or algorithms.
3. **Senior** — trade-offs, failure modes, scaling implications, eventual consistency concerns. Compare alternatives.
4. **Principal** — when to use vs when not to. Boundary decisions. Team/org implications. Cost of adoption.

## Effect v4 code rules

Before writing any Effect code, read `40-tools/effect-v4.md` — it contains:

- The full service definition pattern
- All removed/renamed APIs (v3 → v4 diff table)
- Key idioms (`Effect.fn`, direct error yield, service access)
- Gotchas and import paths
- Pointer to `80-resources/effect-source/` (submodule with LLMS.md, ai-docs/, cookbooks/)

Do not guess Effect v4 API shapes. Check the note first.

## Output format

Follow `90-templates/concept.md` structure:

```markdown
---
title: <topic>
type: concept
created: YYYY-MM-DD
tags: []
---

# <topic>

## What

(one-paragraph plain-language definition)

## Why it matters

## How it works

(junior → mid → senior → principal walkthrough as subsections)

## Examples

(Effect v4 code or plain TypeScript as appropriate)

## Trade-offs

| Pro | Con |
|-----|-----|
|     |     |

## Related

- [[wikilink]]

## References

- 
```

Use `[[wikilink]]` for all internal vault cross-references. Link generously.

## When saving to vault

- `10-concepts/` for timeless concepts (DDD, CQRS, recursion, concurrency models)
- `30-patterns/` for reusable design patterns (repository, saga, outbox)
- `20-languages/` for language-specific idioms
- `40-tools/` for tool/library reference notes
- Do not put version-specific implementation code in `10-concepts/` — it goes in `70-projects/`
- Frontmatter `type` must match folder: `concept`, `pattern`, `language`, `tool`
