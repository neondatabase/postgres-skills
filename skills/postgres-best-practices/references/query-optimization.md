# Query Optimization Reference

## Contents
- Running EXPLAIN ANALYZE
- Reading execution plans
- Plan node reference
- Join strategy selection
- Common bottlenecks and fixes
- Statistics and the planner
- Configuration tuning knobs

## Running EXPLAIN ANALYZE

Always use this form for real diagnostics:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

| Flag | Purpose |
|------|---------|
| `ANALYZE` | Actually executes the query (required for actual times/rows) |
| `BUFFERS` | Shows shared/local buffer hits and reads (I/O insight). **PG18+**: included by default with `ANALYZE` |
| `FORMAT TEXT` | Human-readable output (default) |

Additional EXPLAIN options:

| Flag | PG Version | Purpose |
|------|-----------|---------|
| `GENERIC_PLAN` | PG16+ | Show plan for parameterized queries (with `$1` params) without executing |
| `MEMORY` | PG17+ | Report planner memory usage |
| `SERIALIZE` | PG17+ | Show cost of converting result data for network transmission |

**PG18 EXPLAIN improvements:**
- `BUFFERS` is now included automatically with `ANALYZE` — no need to specify it separately
- Index scan nodes report the number of index lookups
- Memory/disk usage shown for Materialize, Window Aggregate, and CTE Scan nodes
- Fractional row counts in output (instead of rounding to integers)
- Disabled nodes are explicitly indicated in output

For queries that modify data, wrap in a transaction and rollback:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) DELETE FROM orders WHERE ...;
ROLLBACK;
```

### Key Metrics in Output

```
Seq Scan on orders  (cost=0.00..1520.00 rows=50000 width=64)
                     (actual time=0.012..15.432 rows=48573 loops=1)
  Buffers: shared hit=520 read=200
```

- **cost**: Estimated startup..total cost (arbitrary planner units)
- **rows**: Estimated row count
- **actual time**: Real time in ms (startup..total)
- **actual rows**: Real row count — compare to estimate
- **loops**: How many times this node executed
- **Buffers shared hit**: Pages found in cache
- **Buffers shared read**: Pages read from disk

**Critical check**: If estimated `rows` and actual `rows` differ by 10x+, statistics are stale or the planner is misestimating.

## Reading Execution Plans

Read bottom-up, innermost to outermost. Each node:
1. Scans or receives input rows
2. Applies filtering or transformation
3. Passes rows to the parent node

### Identifying the Bottleneck

1. Find the node with the highest `actual time` (total, not startup)
2. Check if actual rows >> estimated rows (bad statistics)
3. Check `Buffers: shared read` — high reads = cold cache or missing index
4. Look for `Rows Removed by Filter` — high values mean the scan is too broad

## Plan Node Reference

### Scan Nodes

| Node | Meaning | When it's a problem |
|------|---------|-------------------|
| `Seq Scan` | Full table scan | On large tables when only a few rows match |
| `Index Scan` | B-tree lookup + heap fetch | Normal and expected |
| `Index Only Scan` | B-tree lookup, no heap fetch | Best case — all columns in index |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Index lookup → bitmap → heap | Good for medium selectivity |
| `CTE Scan` | Reads from materialized CTE | Check if CTE materialization is needed |

**When is Seq Scan OK?**
- Table is small (< few thousand rows)
- Query returns > 10-20% of rows
- No usable index exists and adding one isn't warranted

### Join Nodes

| Node | How it works | Best for |
|------|-------------|----------|
| `Nested Loop` | For each outer row, scan inner | Small outer set + indexed inner |
| `Hash Join` | Build hash table on inner, probe with outer | Medium-large equijoins, enough work_mem |
| `Merge Join` | Both inputs sorted, merge | Pre-sorted inputs or sorted output needed |

**Nested Loop red flags**: high outer row count with no index on the inner side.

**Hash Join red flags**: `Batches: N` where N > 1 means hash table spilled to disk (increase `work_mem`).

### Sort and Aggregate Nodes

| Node | Notes |
|------|-------|
| `Sort` | Check `Sort Method: external merge` = disk spill (increase `work_mem`) |
| `HashAggregate` | Groups via hash table — check for disk spill batches |
| `GroupAggregate` | Groups pre-sorted input — needs sorted input |
| `Incremental Sort` | Sorts remaining columns when leading columns already sorted |

### Other Nodes

| Node | Notes |
|------|-------|
| `Materialize` | Caches a subplan's output for re-scan |
| `Memoize` (PG14+) | Caches parameterized nested loop inner-side results. Look for `Hits: N Misses: N` — high hit ratio means effective caching. Poor cache ratio may indicate high cardinality on the join key |
| `Gather` / `Gather Merge` | Collects results from parallel workers |
| `Append` | Concatenates results (UNION ALL, partitioned tables) |
| `SubPlan` | Correlated subquery — potentially executed per row |

## Common Bottlenecks and Fixes

### Sequential Scan on Large Table

**Symptom**: `Seq Scan` with `Rows Removed by Filter: 999000` (scanned 1M, kept 1K)

**Fixes**:
1. Add an index on the WHERE clause columns
2. If query returns > 20% of rows, seq scan may actually be optimal
3. Check `enable_seqscan = on` isn't masking the real issue

### Nested Loop with High Row Count

**Symptom**: `Nested Loop (actual loops=50000)` with inner `Seq Scan`

**Fixes**:
1. Add index on the inner table's join column
2. If both sides are large, the planner should choose hash/merge join — check statistics

### Sort Spilling to Disk

**Symptom**: `Sort Method: external merge Disk: 125MB`

**Fixes**:
1. Increase `work_mem` (per-operation, not global): `SET work_mem = '256MB';`
2. Add an index matching the ORDER BY to avoid sorting entirely
3. If in a CTE or subquery, consider whether the sort is necessary

### Bad Row Estimates

**Symptom**: Estimated rows: 1, Actual rows: 500,000

**Fixes**:
1. `ANALYZE table_name;` — refresh statistics
2. Increase `default_statistics_target` for columns with skewed distributions:
   ```sql
   ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
   ANALYZE orders;
   ```
3. Create extended statistics for correlated columns:
   ```sql
   CREATE STATISTICS orders_status_date (dependencies)
       ON status, created_at FROM orders;
   ANALYZE orders;
   ```

### Correlated SubPlan Executed Per Row

**Symptom**: `SubPlan` node with `loops=100000`

**Fix**: Rewrite as a JOIN or lateral join:

```sql
-- Bad: correlated subquery
SELECT *, (SELECT name FROM departments d WHERE d.id = e.dept_id) AS dept_name
FROM employees e;

-- Good: join
SELECT e.*, d.name AS dept_name
FROM employees e
JOIN departments d ON d.id = e.dept_id;
```

### PG18 Optimizer Improvements

**B-tree skip scan (PG18+)**: Multi-column B-tree indexes can now be used even when there are no restrictions on the leading columns. Previously a composite index on `(tenant_id, created_at)` was useless for `WHERE created_at > '2024-01-01'` without a `tenant_id` filter. PG18 can skip through distinct `tenant_id` values — eliminating many cases where you needed a separate single-column index.

**Self-join elimination**: The optimizer automatically removes unnecessary self-joins. Controlled by `enable_self_join_elimination`.

**IN to ANY conversion**: `WHERE x IN (VALUES ...)` is converted to `x = ANY(...)` for better use of optimizer statistics.

**OR-clause to array transformation**: OR clauses like `WHERE x = 1 OR x = 2 OR x = 3` are transformed to arrays for faster index processing.

**DISTINCT reordering**: Keys in `SELECT DISTINCT` can be reordered internally to match an existing index and avoid sorting. Controlled by `enable_distinct_reordering`.

**Improved partition planning**: More efficient planning for queries accessing many partitions, with reduced memory usage. Partitionwise joins allowed in more cases.

## Statistics and the Planner

### Manual ANALYZE

```sql
-- Analyze one table
ANALYZE orders;

-- Analyze specific columns
ANALYZE orders(status, created_at);
```

Autovacuum runs ANALYZE automatically, but after bulk loads or major changes, run it manually.

### Extended Statistics

For correlated columns that the planner estimates independently:

```sql
-- Functional dependency: knowing city tells you the state
CREATE STATISTICS city_state_dep (dependencies) ON city, state FROM addresses;

-- N-distinct: correct group count estimates
CREATE STATISTICS city_state_ndist (ndistinct) ON city, state FROM addresses;

-- MCV lists: track most common value combinations
CREATE STATISTICS city_state_mcv (mcv) ON city, state FROM addresses;

ANALYZE addresses;
```

### Checking Current Statistics

```sql
SELECT
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    correlation  -- physical vs. logical ordering (affects index scan cost)
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

## Configuration Tuning Knobs

Except for `shared_buffers`, these parameters can be overridden in the current session with `SET` for workload-specific testing. `shared_buffers` is allocated at server start; persisting a change in `postgresql.conf` or with `ALTER SYSTEM` requires a restart before it takes effect.

| Parameter | Default | When to increase |
|-----------|---------|-----------------|
| `shared_buffers` | 128MB | Server-wide, restart-required setting; start around 25% of total RAM and validate for the workload. See performance-diagnostics for cache hit rate queries |
| `work_mem` | 4MB | Sort/hash spilling to disk |
| `maintenance_work_mem` | 64MB | Slow VACUUM, CREATE INDEX, ALTER TABLE |
| `effective_cache_size` | 4GB | Set to ~75% of total RAM (hint to planner, no allocation) |
| `random_page_cost` | 4.0 | SSD storage → set to 1.1-1.5 |
| `effective_io_concurrency` | 1 (16 in PG18) | SSD storage → set to 200 |
| `default_statistics_target` | 100 | Bad estimates on skewed columns → 500-1000 |
| `max_parallel_workers_per_gather` | 2 | Large analytical queries benefit from more parallelism |

### SSD-Optimized Settings

```sql
SET random_page_cost = 1.1;
-- Some platforms without posix_fadvise only support 0.
DO $$
BEGIN
    PERFORM set_config('effective_io_concurrency', '200', false);
EXCEPTION
    WHEN invalid_parameter_value THEN
        PERFORM set_config('effective_io_concurrency', '0', false);
END
$$;
SET effective_cache_size = '24GB';  -- adjust to your server
```

These make the planner more willing to use index scans when measurements show SSD random reads are fast. Some platforms without asynchronous prefetch support cap `effective_io_concurrency` at `0`.

### Asynchronous I/O (PG18+)

PG18 introduces a native async I/O subsystem that improves sequential scan, bitmap heap scan, and VACUUM performance.

| Parameter | Default (PG18) | Purpose |
|-----------|----------------|---------|
| `io_method` | platform-dependent | I/O method: `sync`, `io_uring` (Linux), `posix_aio` |
| `io_combine_limit` | 128kB | Max size of combined I/O operations |
| `effective_io_concurrency` | 16 (was 1) | Default raised significantly in PG18 |
| `maintenance_io_concurrency` | 16 (was 10) | Default raised for maintenance operations |

The raised defaults mean PG18 out-of-the-box performance for sequential scans and VACUUM is significantly better than prior versions.
