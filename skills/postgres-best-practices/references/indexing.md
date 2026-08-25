# Indexing Reference

## Contents
- Index types and when to use each
- Composite index column ordering
- Partial indexes
- Covering indexes (INCLUDE)
- Expression indexes
- Index maintenance and bloat
- Finding unused and duplicate indexes

## Index Types

### B-tree (Default)

Supports: `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `IS NOT NULL`

```sql
CREATE INDEX idx_orders_created ON orders(created_at);
```

Best for: equality and range queries on scalar types. The default and most common choice.

### GIN (Generalized Inverted Index)

Supports: containment operators on composite values.

```sql
-- JSONB containment
CREATE INDEX idx_data_gin ON events USING gin(payload);
-- Matches: WHERE payload @> '{"status": "active"}'

-- Array containment
CREATE INDEX idx_tags_gin ON articles USING gin(tags);
-- Matches: WHERE tags @> ARRAY['postgres']

-- Full-text search
CREATE INDEX idx_fts ON articles USING gin(to_tsvector('english', body));
-- Matches: WHERE to_tsvector('english', body) @@ to_tsquery('postgres & index')

-- Trigram (fuzzy text search)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_name_trgm ON users USING gin(name gin_trgm_ops);
-- Matches: WHERE name ILIKE '%pattern%'
```

**Operator class tip**: For JSONB, `jsonb_path_ops` is 2-3x smaller than the default `jsonb_ops` but only supports `@>` (containment). Use it when you only need containment queries:

```sql
CREATE INDEX idx_events_gin_path ON events USING gin(payload jsonb_path_ops);
-- Only supports: WHERE payload @> '{"type": "click"}'
-- Does NOT support: WHERE payload ? 'type'
```

**Stored generated column for FTS**:

```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;
CREATE INDEX idx_articles_search ON articles USING gin(search_vector);
-- Query: WHERE search_vector @@ to_tsquery('english', 'postgres & optimization')
```

GIN indexes are larger and slower to update than B-tree but excel at multi-valued containment queries.

**Parallel GIN builds (PG18+)**: GIN indexes can now be built in parallel, significantly speeding up index creation on large tables.

### GiST (Generalized Search Tree)

Supports: overlap, containment, nearest-neighbor on geometric, range, and full-text types.

```sql
-- Range overlap
CREATE INDEX idx_booking_range ON bookings USING gist(during);
-- Matches: WHERE during && '[2024-01-01, 2024-02-01)'

-- Used in exclusion constraints
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings
    ADD CONSTRAINT bookings_no_overlap
    EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

### BRIN (Block Range Index)

Tiny index for naturally ordered data (e.g., append-only timestamp columns).

```sql
CREATE INDEX idx_events_ts_brin ON events USING brin(occurred_at);
```

BRIN stores min/max per block range. Effective when physical row order correlates with column value. Very small (~0.1% of B-tree size) but less precise — may read extra blocks.

### Hash

Only supports `=`. Rarely better than B-tree.

```sql
CREATE INDEX idx_session_hash ON sessions USING hash(session_token);
```

## Composite Index Column Ordering

Order matters. The index is a sorted tree, leftmost column first.

**Rule of thumb**: equality columns first, then range/sort columns.

```sql
-- Query: WHERE tenant_id = 1 AND created_at > '2024-01-01' ORDER BY created_at
CREATE INDEX idx_tenant_created ON orders(tenant_id, created_at);
```

The index skips to `tenant_id = 1` (equality), then range-scans `created_at` in order.

**Leading column rule**: A composite index on `(a, b, c)` can serve queries on:
- `a` alone
- `a, b`
- `a, b, c`

But NOT `b` alone or `c` alone (pre-PG18).

**B-tree skip scan (PG18+)**: PG18 can skip through distinct values of leading columns, so an index on `(a, b, c)` can now serve queries on `b` or `c` alone — by scanning each distinct `a` value. This works best when the leading column has low cardinality (few distinct values). It eliminates many cases where you previously needed a separate single-column index.

## Partial Indexes

Index only the rows that matter. Smaller index = faster lookups, less maintenance.

```sql
-- Only index active orders (95% of queries filter on active)
CREATE INDEX idx_orders_active ON orders(customer_id)
    WHERE status = 'active';

-- Only index non-null values
CREATE INDEX idx_orders_shipped ON orders(shipped_at)
    WHERE shipped_at IS NOT NULL;
```

The query's WHERE clause must match (or imply) the index predicate for the planner to use it.

## Covering Indexes (INCLUDE)

Add non-key columns to enable index-only scans without bloating the B-tree structure.

```sql
-- Query: SELECT email, name FROM users WHERE email = ?
CREATE UNIQUE INDEX idx_users_email ON users(email) INCLUDE (name);
```

The `name` column is stored in the index leaf pages but not in the B-tree structure. This enables an index-only scan (no heap fetch) without affecting index ordering or uniqueness.

## Expression Indexes

Index the result of an expression or function.

```sql
-- Case-insensitive email lookup
CREATE UNIQUE INDEX idx_users_email_lower ON users(lower(email));
-- Query must match: WHERE lower(email) = lower($1)

-- JSONB field extraction
CREATE INDEX idx_events_type ON events((payload->>'type'));
-- Query: WHERE payload->>'type' = 'click'

-- Date truncation
-- timestamptz must be made timezone-independent for an immutable expression
CREATE INDEX idx_orders_month
    ON orders(date_trunc('month', created_at AT TIME ZONE 'UTC'));
```

The query must use the same expression for the planner to match the index.

## Indexes on Partitioned Tables

Indexes defined on a partitioned parent table are **automatically created on all existing and future child partitions**.

```sql
-- Create index on the partitioned parent — propagates to all partitions
CREATE INDEX idx_events_type ON events(occurred_at, (payload->>'type'));
```

### Key Rules

- **Partition key in unique indexes**: Any UNIQUE or PRIMARY KEY index on a partitioned table must include all partition key columns.

```sql
-- Partitioned by occurred_at — PK must include it
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY,
    occurred_at timestamptz NOT NULL,
    payload jsonb
) PARTITION BY RANGE (occurred_at);

-- This works:
ALTER TABLE events ADD PRIMARY KEY (id, occurred_at);

-- This fails: partition key 'occurred_at' not in the constraint
-- ALTER TABLE events ADD PRIMARY KEY (id);
```

- **Per-partition indexes**: You can also create indexes on individual partitions for partition-specific optimizations. These won't propagate to other partitions.

```sql
-- Extra index only on the hot partition
CREATE INDEX idx_events_q1_status ON events_2024_q1((payload->>'status'));
```

- **CONCURRENTLY on partitioned tables**: PostgreSQL does not support `CREATE INDEX CONCURRENTLY` directly on a partitioned parent. Create an invalid parent index with `ON ONLY`, build matching indexes concurrently on each partition, then attach them:

```sql
CREATE INDEX idx_events_customer
    ON ONLY events ((payload->>'customer_id'));

CREATE INDEX CONCURRENTLY idx_events_q1_customer
    ON events_2024_q1 ((payload->>'customer_id'));

ALTER INDEX idx_events_customer
    ATTACH PARTITION idx_events_q1_customer;
```

Repeat the concurrent build and attach steps for every partition. The parent index becomes valid after all partition indexes are attached. If a child build fails, drop or rebuild that invalid child index before attaching it.

- **Index-only scans**: Work across partitions. The planner prunes irrelevant partitions first, then uses index-only scans on the remaining ones.

- **REINDEX on partitioned tables**: `REINDEX TABLE` on a partitioned table reindexes all partitions.

```sql
REINDEX TABLE CONCURRENTLY events;
```

## Concurrent Index Creation

For production use, always create indexes concurrently to avoid locking writes:

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
```

Caveats:
- Takes longer (two table scans instead of one)
- Cannot run inside a transaction block
- If it fails, leaves an `INVALID` index — drop and retry
- Check for invalid indexes: `SELECT * FROM pg_index WHERE NOT indisvalid;`

## Index Maintenance

### Reindexing Bloated Indexes

```sql
-- Concurrent reindex
REINDEX INDEX CONCURRENTLY idx_orders_customer;

-- Or reindex all indexes on a table
REINDEX TABLE CONCURRENTLY orders;
```

### Monitoring Index Size

```sql
SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan AS scans,
    idx_tup_read AS tuples_read
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Finding Unused Indexes

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey'   -- exclude primary keys
    AND indexrelname NOT LIKE '%unique%' -- exclude unique constraints
ORDER BY pg_relation_size(indexrelid) DESC;
```

Reset statistics after a representative period: `SELECT pg_stat_reset();`

## Finding Duplicate Indexes

```sql
SELECT
    array_agg(indexrelid::regclass) AS indexes,
    indrelid::regclass AS table_name,
    indkey AS column_numbers
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;
```

Also check for indexes that are a prefix of another:
- `(a)` is redundant if `(a, b)` exists
- `(a, b)` is NOT redundant if `(a, b, c)` exists and you need index-only scans on just `(a, b)`

## Index Selection Decision Tree

1. **What operator?**
   - `=`, `<`, `>`, `BETWEEN` → B-tree
   - `@>`, `?`, `&&` on jsonb/array → GIN
   - `@@` full-text → GIN
   - `ILIKE '%x%'` → GIN + pg_trgm
   - Range/geometric overlap → GiST
   - Naturally ordered append-only → BRIN

2. **How many rows match?** If > 10-20% of table, index may not help (seq scan is cheaper).

3. **Can you narrow the index?** Use a partial index if most queries filter on a subset.

4. **Do you need index-only scans?** Add `INCLUDE` columns.

5. **Is the column an expression?** Use an expression index.
