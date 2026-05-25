# SQL Indexing & Query Performance

> **Session:** May 26, 2026
> **Topics:** What is an Index → Hash Maps vs B-Trees → B-Tree Complexity → Write Overhead of Indexes → Selectivity → When to Index
> **Tags:** [sql, postgresql, indexes, b-tree, performance, query-optimization, database]

---

## What is an Index?

An index is a separate data structure that acts as a **map** for a column — instead of scanning every row in a table, the database jumps directly to the relevant rows.

- **Without index:** Full table scan → O(n)
- **With index:** Jump directly → O(log n) for B-tree

---

## Why Not Hash Maps?

Hash maps give O(1) lookups but **don't preserve order**. This means they break for:

```sql
WHERE price > 2000        -- range query
WHERE name LIKE 'Kar%'    -- prefix query
ORDER BY created_at       -- sorting
```

Hash indexes only work for **exact lookups**:

```sql
WHERE slug = 'red-silk-saree'  -- ✅ hash works here
```

---

## B-Tree — The Default Index

PostgreSQL uses **B-trees** by default. They're sorted and balanced, supporting:
- Exact lookups
- Range queries (`>`, `<`, `BETWEEN`)
- Prefix matches (`LIKE 'abc%'`)
- Ordering (`ORDER BY`)

**Complexity:** O(log n)

### Why log n matters

| Rows | Full Scan | B-Tree Steps |
|------|-----------|--------------|
| 1,000,000 | 1,000,000 | ~20 |
| 10,000,000 | 10,000,000 | ~23 |

For 10 million rows, a B-tree finds your value in **~23 steps**.

---

## The Write Overhead Tradeoff

Every `INSERT`, `UPDATE`, or `DELETE` must update **all indexes** on that table.

- 1 index → 1 update per write
- 10 indexes → 10 updates per write

**Rule of thumb:**
- Read-heavy tables → more indexes are fine
- Write-heavy tables → be selective with indexes

---

## Selectivity — The Most Important Concept

**Selectivity** = how much a query narrows down the result set.

| Query | Rows Returned | Selectivity | Index Useful? |
|-------|--------------|-------------|---------------|
| `WHERE slug = 'red-saree'` | 1 | Very High | ✅ Yes |
| `WHERE price > 2000` | ~30% | Medium | ✅ Yes |
| `WHERE category = 'sarees'` | ~40% | Low | ❌ Probably not |

> If an index still returns 40% of the table, PostgreSQL might just do a full scan anyway.

---

## When to Index — Decision Checklist

1. ✅ Is this column frequently used in `WHERE`, `ORDER BY`, or `JOIN`?
2. ✅ Does it have **high selectivity** (narrows down results significantly)?
3. ✅ Is the table more **read-heavy** than write-heavy?
4. ❌ Avoid indexing columns with very few distinct values (e.g. `status` with 3 values)

---

## Unique Constraints = Free Index

When you add a `UNIQUE` constraint, PostgreSQL **automatically creates a B-tree index** under the hood:

```sql
-- This creates an index automatically
ALTER TABLE products ADD CONSTRAINT products_slug_unique UNIQUE(slug);

-- No need to also do this separately
CREATE INDEX idx_products_slug ON products(slug); -- redundant!
```

---

## Practical Example — Boutique `products` Table

| Column | Index? | Reason |
|--------|--------|--------|
| `slug` | ✅ UNIQUE constraint | Exact lookup for product pages, must be unique |
| `price` | ✅ B-tree | Range queries (`WHERE price < 2000`) |
| `size` | ✅ B-tree | Exact filter, high selectivity |
| `color` | ✅ B-tree | Exact filter, high selectivity |
| `category` | ⚠️ Maybe not | Low selectivity if few categories |

---

## Quick Reference

```sql
-- Default B-tree index
CREATE INDEX idx_products_price ON products(price);

-- Hash index (only for exact lookups)
CREATE INDEX idx_products_slug ON products USING hash(slug);

-- Unique constraint (auto-creates index)
ALTER TABLE products ADD CONSTRAINT products_slug_unique UNIQUE(slug);

-- Check existing indexes in PostgreSQL
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'products';
```
