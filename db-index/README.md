# Database Indexing Guide

Quick reference for SQL index strategies and best practices.

## Contents

| File | Description |
|------|-------------|
| `01-index-best-practices.md` | Single vs composite indexes, decision framework |
| `02-table-scan-vs-index-merge.md` | I/O explained, meeting scheduler example |
| `03-clustered-vs-non-clustered-index.md` | Clustered vs non-clustered simplified |

## Key Takeaways

- **Composite indexes** > relying on index merge
- Design indexes for **queries**, not tables
- **Equality columns first**, range columns last
- One table = **one clustered index** only

