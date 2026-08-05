---
title: Effect Source Reference
type: resource
created: 2026-08-05
tags: [effect, reference]
---

# Effect Source Reference

The `effect-source/` directory is a git submodule pointing to [Effect-TS/effect](https://github.com/Effect-TS/effect).

Used as a canonical reference when writing Effect examples in this vault — no guessing from v2/v3 memory.

## Update manually

```sh
cd ~/dev/personal/knowledge
git submodule update --remote --merge 80-resources/effect-source
git add 80-resources/effect-source
git commit -m "chore: update effect submodule"
```

## Auto-update

A `post-merge` hook runs the above automatically after every `git pull` on the vault.

## Key files

| File | What |
|---|---|
| `packages/effect/src/Effect.ts` | Core Effect type and combinators |
| `packages/effect/src/Data.ts` | `TaggedError`, `TaggedClass`, `Class` |
| `packages/effect/src/Brand.ts` | Branded types |
| `packages/effect/src/Context.ts` | `Context.Service`, dependency injection |
| `packages/effect/src/Layer.ts` | `Layer` — wiring implementations |
| `packages/effect/src/Schema.ts` | Runtime validation + serialization |
| `packages/effect/src/Match.ts` | Pattern matching |
| `packages/sql/` | SQL integrations (Drizzle, PG, SQLite) |
| `packages/platform-node/` | Node.js platform services |
