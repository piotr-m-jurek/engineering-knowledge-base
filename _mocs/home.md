---
title: Home
type: moc
created: 2026-08-05
---

# Programming Knowledge Vault

> Zettelkasten-style knowledge base. Atomic notes, heavy linking, MOCs as entry points.

## Entry Points

- [[_mocs/concepts]] — mental models, CS fundamentals, algorithms
- [[_mocs/languages]] — TypeScript, Kotlin, Swift, Rust, etc.
- [[_mocs/patterns]] — design patterns, architectural patterns, idioms
- [[_mocs/tools]] — CLIs, editors, build tools, testing frameworks
- [[_mocs/architectures]] — system design, distributed systems, frontend arch
- [[_mocs/decisions]] — ADRs and technology choices
- [[_mocs/projects]] — per-project knowledge clusters

## Recent Notes

```dataview
TABLE file.mtime AS "Modified", type
FROM "" AND !path("_mocs") AND !path("90-templates")
SORT file.mtime DESC
LIMIT 10
```

## Inbox (needs processing)

```dataview
LIST
FROM "00-inbox"
SORT file.ctime DESC
```

## Stats

```dataview
TABLE length(rows) AS "Count"
FROM "" AND !path("_mocs") AND !path("90-templates") AND !path(".obsidian")
GROUP BY type
SORT length(rows) DESC
```
