# Clustered Index vs Non-Clustered Index

Explain clustered index vs non-clustered index as simple as possible.

---

## The Simplest Explanation

| Type | What It Is |
|------|------------|
| **Clustered Index** | The table itself, sorted by the index key |
| **Non-Clustered Index** | A separate "lookup table" pointing to rows |

---

## Visual Analogy: A Book

```
┌─────────────────────────────────────────────────────────────────┐
│  CLUSTERED INDEX = The book's page order                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A dictionary is sorted A-Z                                     │
│                                                                 │
│     Page 1      Page 2      Page 3      Page 4                  │
│    ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐                   │
│    │ Ant  │   │ Bear │   │ Cat  │   │ Dog  │                   │
│    │ Apple│   │ Bird │   │ Cow  │   │ Duck │                   │
│    │ Art  │   │ Book │   │ Cup  │   │      │                   │
│    └──────┘   └──────┘   └──────┘   └──────┘                   │
│                                                                 │
│  ✓ Data IS the index (physically sorted)                        │
│  ✓ Only ONE way to sort a book                                  │
│  ✓ Finding "Cat" = go directly to the C section                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  NON-CLUSTERED INDEX = The index at the back of the book        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A textbook's index:                                            │
│                                                                 │
│     INDEX (at back)              ACTUAL PAGES                   │
│    ┌─────────────────┐          ┌──────┐                       │
│    │ Einstein ... 42 │ ──────▶  │ p.42 │ (content here)        │
│    │ Gravity .... 15 │ ──────▶  │ p.15 │                       │
│    │ Newton ..... 38 │ ──────▶  │ p.38 │                       │
│    │ Physics .... 7  │ ──────▶  │ p.7  │                       │
│    └─────────────────┘          └──────┘                       │
│                                                                 │
│  ✓ Index is SEPARATE from data                                  │
│  ✓ Can have MANY indexes (by topic, by author, etc.)            │
│  ✓ Finding "Newton" = look up index → go to page 38             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## In Database Terms

```
┌─────────────────────────────────────────────────────────────────┐
│  CLUSTERED INDEX                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Table data is PHYSICALLY SORTED by the index key               │
│                                                                 │
│  CREATE TABLE users (                                           │
│      id INT PRIMARY KEY,  ← Clustered index (by default)        │
│      name VARCHAR(100),                                         │
│      email VARCHAR(255)                                         │
│  );                                                             │
│                                                                 │
│  Disk storage:                                                  │
│  ┌────────────────────────────────────────┐                    │
│  │ id=1 │ John  │ john@mail.com           │ ← Row 1            │
│  │ id=2 │ Alice │ alice@mail.com          │ ← Row 2            │
│  │ id=3 │ Bob   │ bob@mail.com            │ ← Row 3            │
│  │ id=4 │ Carol │ carol@mail.com          │ ← Row 4            │
│  └────────────────────────────────────────┘                    │
│       ↑                                                         │
│       Data stored IN ORDER of id                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  NON-CLUSTERED INDEX                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Separate structure with POINTERS to actual rows                │
│                                                                 │
│  CREATE INDEX idx_name ON users(name);                          │
│                                                                 │
│  Index structure:          Table (unchanged):                   │
│  ┌─────────┬─────────┐    ┌────────────────────────────────┐   │
│  │  name   │ pointer │    │ id=1 │ John  │ john@mail.com   │   │
│  ├─────────┼─────────┤    │ id=2 │ Alice │ alice@mail.com  │   │
│  │ Alice   │ ───────────▶ │ id=3 │ Bob   │ bob@mail.com    │   │
│  │ Bob     │ ───────────▶ │ id=4 │ Carol │ carol@mail.com  │   │
│  │ Carol   │ ───────────▶ └────────────────────────────────┘   │
│  │ John    │ ───────────▶                                      │
│  └─────────┴─────────┘                                         │
│       ↑                                                         │
│       Sorted by name, points to actual row location             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Differences

| Aspect | Clustered | Non-Clustered |
|--------|-----------|---------------|
| **How many?** | Only **1** per table | **Many** per table |
| **Data storage** | Index IS the data | Separate from data |
| **Lookup speed** | Faster (no extra hop) | Slower (must follow pointer) |
| **Insert/Update** | Slower (must maintain order) | Faster |
| **Size** | No extra space | Extra space for index |

---

## One-Liner Summary

> **Clustered** = Data sorted on disk by this key (the table itself)
> 
> **Non-Clustered** = Separate lookup list with pointers to rows

---

## Quick SQL Example

```sql
-- Clustered: PRIMARY KEY is clustered by default (SQL Server/MySQL InnoDB)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,        -- ← Clustered index
    customer_id INT,
    total DECIMAL(10,2)
);

-- Non-clustered: Any additional index you create
CREATE INDEX idx_customer ON orders(customer_id);  -- ← Non-clustered
```

**When you query:**

```sql
-- Uses clustered index (direct access)
SELECT * FROM orders WHERE order_id = 100;

-- Uses non-clustered index (lookup → then fetch row)
SELECT * FROM orders WHERE customer_id = 5;
```

