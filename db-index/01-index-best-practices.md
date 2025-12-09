# What is the best practice to define indexes for a table?

Should we focus on multiple indexes of single columns, or combined index on those most queried, or should we do both? I heard that merging 2 or more indexes is time consuming for some DBMS such as SQL Server... I'm not sure is there any best practice, and does it have a framework to determine exactly the most desirable indexes should be created to balance between memory and performance?

---

## The Short Answer

**Composite indexes on your most common query patterns** are generally preferred over relying on index merges. But it's not black and white—here's the framework:

---

## 1. Index Merge is Expensive (You're Right!)

| Database | Index Merge Behavior |
|----------|---------------------|
| **SQL Server** | Uses "Index Intersection" - creates bitmaps, merges them. Can be slower than a table scan for low selectivity |
| **PostgreSQL** | "Bitmap Index Scan" - better than SQL Server but still has overhead |
| **MySQL** | "Index Merge" - often disabled by optimizer because it's slow |

**Why it's slow:**
- Each index lookup returns row IDs
- DBMS must sort/intersect these ID sets
- Then fetch actual rows (random I/O)
- Often worse than a single composite index or even a table scan

---

## 2. The Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│           INDEX DECISION FRAMEWORK                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  STEP 1: Identify your query patterns                       │
│  ─────────────────────────────────────                      │
│  • What WHERE clauses appear most often?                    │
│  • What ORDER BY / GROUP BY are common?                     │
│  • What JOINs happen frequently?                            │
│                                                             │
│  STEP 2: Prioritize by frequency × cost                     │
│  ─────────────────────────────────────                      │
│  Score = (query frequency) × (current cost without index)   │
│                                                             │
│  STEP 3: Design composite indexes for top patterns          │
│  ─────────────────────────────────────                      │
│  • Put equality columns first (WHERE x = ?)                 │
│  • Put range columns last (WHERE y > ?)                     │
│  • Include ORDER BY columns if possible                     │
│                                                             │
│  STEP 4: Check for index reuse (leftmost prefix rule)       │
│  ─────────────────────────────────────                      │
│  Index(A, B, C) serves: A, A+B, A+B+C                       │
│  Can one composite replace multiple single-col indexes?     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Best Practices

### ✅ DO: Create Composite Indexes for Common Query Patterns

```sql
-- If you frequently query:
SELECT * FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC

-- Create:
CREATE INDEX idx_orders_cust_status_created 
ON orders(customer_id, status, created_at DESC);
```

### ✅ DO: Use Single-Column Indexes for Columns Queried Independently

```sql
-- If email is searched alone often:
CREATE INDEX idx_users_email ON users(email);
```

### ✅ DO: Consider Covering Indexes (Include All Selected Columns)

```sql
-- PostgreSQL/SQL Server: INCLUDE clause avoids table lookup
CREATE INDEX idx_covering ON orders(customer_id) 
INCLUDE (total, status);  -- These columns are in the leaf nodes
```

### ❌ DON'T: Create Redundant Indexes

```sql
-- BAD: These are redundant
CREATE INDEX idx_a ON t(a);
CREATE INDEX idx_ab ON t(a, b);  -- This already covers queries on just (a)

-- GOOD: Just keep the composite
CREATE INDEX idx_ab ON t(a, b);
```

### ❌ DON'T: Over-Index

Each index:
- Slows down INSERT/UPDATE/DELETE (must update all indexes)
- Consumes disk space and memory
- May confuse the query optimizer

---

## 4. Formal Framework: The "Index Selection Problem"

Database vendors provide tools based on workload analysis:

| Tool | Database | What It Does |
|------|----------|--------------|
| **Database Engine Tuning Advisor (DTA)** | SQL Server | Analyzes workload, recommends indexes |
| **pg_stat_statements + HypoPG** | PostgreSQL | Track queries + test hypothetical indexes |
| **MySQL Enterprise Query Analyzer** | MySQL | Workload-based recommendations |
| **EXPLAIN ANALYZE** | All | Manual analysis per query |

### DIY Approach:

```sql
-- PostgreSQL: Find slow queries without good indexes
SELECT query, calls, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time * calls DESC
LIMIT 20;

-- Then use EXPLAIN ANALYZE on each to see if indexes help
```

---

## 5. My Recommended Strategy

```
┌────────────────────────────────────────────────────────────────┐
│  PRACTICAL INDEXING STRATEGY                                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. PRIMARY KEY           → Always (automatic)                 │
│                                                                │
│  2. FOREIGN KEYS          → Always index FK columns            │
│                                                                │
│  3. TOP 5-10 QUERIES      → Composite indexes tailored to each │
│                                                                │
│  4. UNIQUE CONSTRAINTS    → Columns that must be unique        │
│                                                                │
│  5. SINGLE-COLUMN         → Only for columns frequently        │
│                              queried ALONE and not covered     │
│                              by composite index prefix         │
│                                                                │
│  6. AVOID                 → Low-selectivity columns alone      │
│                              (e.g., boolean, status with       │
│                              only 3-4 values)                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 6. Memory vs Performance Trade-off

**Rule of thumb:**
- Total index size should ideally fit in RAM (buffer pool)
- If indexes exceed RAM, you get disk I/O on index lookups

**Calculate index overhead:**
```sql
-- PostgreSQL: Check index sizes
SELECT 
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- MySQL:
SELECT 
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE stat_name = 'size';
```

---

## TL;DR

| Approach | When to Use |
|----------|-------------|
| **Composite index** | Queries with multiple columns in WHERE/ORDER BY (preferred) |
| **Single-column indexes** | Column queried alone, not covered by existing composite |
| **Both** | When query patterns genuinely differ |
| **Rely on index merge** | Almost never by design—it's a fallback, not a strategy |

**The golden rule:** Design indexes for your queries, not your tables. Start with query patterns, work backward to indexes.

