---
title: Decisions MOC
type: moc
created: 2026-08-05
---

# Architecture Decision Records

```dataview
TABLE status, tags, file.mtime AS "Modified"
FROM "60-decisions"
SORT file.mtime DESC
```

## By Status

```dataview
TABLE file.link AS "ADR", tags
FROM "60-decisions"
GROUP BY status
```
