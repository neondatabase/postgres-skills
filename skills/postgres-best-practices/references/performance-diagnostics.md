# Performance Diagnostics Reference

## Contents
- Essential pg_stat views
- Table health diagnostics
- Index health diagnostics
- Active query analysis
- Lock analysis
- VACUUM and bloat
- Connection management
- pg_stat_statements setup and queries

## Essential pg_stat Views

| View | What it tells you |
|------|------------------|
| `pg_stat_user_tables` | Seq scans, index scans, row counts, dead tuples, last vacuum/analyze |
| `pg_stat_user_indexes` | Index usage counts, tuple reads |
| `pg_stat_activity` | Currently running queries, wait events, state |
| `pg_stat_statements` | Top queries by time, calls, rows (extension) |
| `pg_stat_bgwriter` | Checkpoint frequency, buffer allocation |
| `pg_stat_io` (PG16+) | I/O statistics by backend type. PG18+ adds byte-level columns and per-backend stats |
| `pg_locks` | Current locks held and awaited |

## Table Health Diagnostics

### Tables with Most Sequential Scans

```sql
SELECT
    schemaname,
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    CASE WHEN seq_scan + idx_scan > 0
         THEN round(100.0 * idx_scan / (seq_scan + idx_scan), 1)
         ELSE 0 END AS idx_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;
```

High `seq_tup_read` with low `idx_scan_pct` = missing index opportunity.

### Tables with High Dead Tuple Ratio

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup,
    n_dead_tup,
    CASE WHEN n_live_tup > 0
         THEN round(100.0 * n_dead_tup / n_live_tup, 1)
         ELSE 0 END AS dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

`dead_pct` > 20% = VACUUM is falling behind. Check autovacuum settings.

### Relation Storage Breakdown

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(
        pg_total_relation_size(schemaname || '.' || tablename) -
        pg_relation_size(schemaname || '.' || tablename)
    ) AS index_and_toast_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;
```

This separates heap size from indexes and TOAST; it is not a bloat estimate. For bloat measurements, use the `pgstattuple` extension (requires superuser or elevated privileges):

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders');
-- dead_tuple_percent > 20% = significant bloat
```

## Index Health Diagnostics

### Unused Indexes (Wasting Space and Slowing Writes)

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
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Index Hit Rate (Should Be > 99%)

```sql
SELECT
    sum(idx_blks_hit) AS idx_hit,
    sum(idx_blks_read) AS idx_read,
    CASE WHEN sum(idx_blks_hit + idx_blks_read) > 0
         THEN round(100.0 * sum(idx_blks_hit) / sum(idx_blks_hit + idx_blks_read), 2)
         ELSE 100 END AS hit_rate_pct
FROM pg_statio_user_indexes;
```

### Table Cache Hit Rate (Should Be > 99%)

```sql
SELECT
    sum(heap_blks_hit) AS heap_hit,
    sum(heap_blks_read) AS heap_read,
    CASE WHEN sum(heap_blks_hit + heap_blks_read) > 0
         THEN round(100.0 * sum(heap_blks_hit) / sum(heap_blks_hit + heap_blks_read), 2)
         ELSE 100 END AS hit_rate_pct
FROM pg_statio_user_tables;
```

Hit rate < 99% = consider increasing `shared_buffers` or optimizing queries.

### shared_buffers Tuning

`shared_buffers` is the primary in-memory page cache. Every page read from disk passes through it, and frequently accessed pages stay cached here.

**Sizing rule of thumb**: Start at **25% of total RAM**. Going higher (up to ~40%) can help on read-heavy workloads with large working sets, but beyond that the OS page cache becomes less effective and returns diminish.

```sql
-- Current setting
SHOW shared_buffers;

-- Effective cache (shared_buffers + OS page cache estimate — planner hint only)
SHOW effective_cache_size;
```

**Diagnostic: Is shared_buffers large enough?**

Combine the cache hit rate above with a working set estimate:

```sql
-- Total size of all user tables and indexes
SELECT
    pg_size_pretty(sum(pg_total_relation_size(relid))) AS total_data_size
FROM pg_stat_user_tables;

-- Compare to shared_buffers
SELECT
    setting || ' ' || unit AS shared_buffers,
    pg_size_pretty(setting::bigint * 8192) AS shared_buffers_bytes
FROM pg_settings
WHERE name = 'shared_buffers';
```

If your hot data (frequently accessed tables + their indexes) significantly exceeds `shared_buffers`, the hit rate drops and you'll see more `shared read` in EXPLAIN output.

**When to increase shared_buffers:**
- Table/index cache hit rate consistently below 99%
- `shared read` dominates `shared hit` in EXPLAIN plans for hot queries
- Server has available RAM (check OS isn't swapping)

**When NOT to increase shared_buffers:**
- Low hit rate caused by full table scans (fix with indexes or query changes, not more cache)
- Server is already memory-constrained (each connection also uses `work_mem`, `maintenance_work_mem`, etc.)
- Hit rate is already 99%+ (adding more cache won't help)

**After changing shared_buffers:**
- Requires a server restart (`pg_ctl restart`)
- Update `effective_cache_size` to roughly `shared_buffers + estimated OS page cache` (typically ~75% of total RAM)
- Monitor hit rates for a representative period after the change

**Per-table cache usage** (requires `pg_buffercache` extension):

```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT
    c.relname,
    pg_size_pretty(count(*) * current_setting('block_size')::bigint) AS buffered,
    round(100.0 * count(*) / (SELECT setting::int FROM pg_settings WHERE name = 'shared_buffers'), 1) AS pct_of_cache
FROM pg_buffercache b
JOIN pg_class c ON pg_relation_filenode(c.oid) = b.relfilenode
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;
```

This shows which tables/indexes are consuming the most shared_buffers — useful for identifying cache hogs or verifying that hot tables are actually cached.

### Finding Missing Foreign Key Indexes

FK columns without indexes cause slow JOINs and slow CASCADE deletes:

```sql
SELECT
    c.conrelid::regclass AS table_name,
    c.conname AS constraint_name,
    pg_get_constraintdef(c.oid) AS constraint_definition
FROM pg_constraint c
WHERE c.contype = 'f'
    AND NOT EXISTS (
        SELECT 1 FROM pg_index i
        WHERE i.indrelid = c.conrelid
            AND i.indisvalid
            AND i.indpred IS NULL
            AND i.indnkeyatts >= cardinality(c.conkey)
            AND (
                SELECT array_agg(key_attnum ORDER BY ordinality)
                FROM unnest(i.indkey::smallint[]) WITH ORDINALITY
                    AS keys(key_attnum, ordinality)
                WHERE ordinality <= cardinality(c.conkey)
            ) = c.conkey
    );
```

### pg_stat_io (PG16+)

I/O statistics by backend type and context — helps identify I/O-heavy operations:

```sql
SELECT
    backend_type, object, context,
    reads, read_time,
    writes, write_time,
    hits
FROM pg_stat_io
WHERE reads > 0 OR writes > 0
ORDER BY reads + writes DESC
LIMIT 10;
```

## Active Query Analysis

### Currently Running Queries

```sql
SELECT
    pid,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    left(query, 100) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
    AND pid != pg_backend_pid()
ORDER BY duration DESC;
```

### Long-Running Queries (> 5 minutes)

```sql
SELECT
    pid,
    now() - query_start AS duration,
    usename,
    application_name,
    client_addr,
    left(query, 200) AS query
FROM pg_stat_activity
WHERE state = 'active'
    AND now() - query_start > interval '5 minutes'
    AND pid != pg_backend_pid()
ORDER BY duration DESC;
```

### Cancel or Terminate a Query

```sql
-- Replace 12345 with a PID selected from pg_stat_activity.
-- Graceful cancel (sends a cancel signal)
SELECT pg_cancel_backend(12345);

-- Force terminate (kills the connection)
SELECT pg_terminate_backend(12345);
```

Do not target your current session (`pg_backend_pid()`). Prefer cancellation first; terminate only when cancellation does not resolve the problem.

## Lock Analysis

### Blocked Queries and What's Blocking Them

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
    AND kl.database IS NOT DISTINCT FROM bl.database
    AND kl.relation IS NOT DISTINCT FROM bl.relation
    AND kl.page IS NOT DISTINCT FROM bl.page
    AND kl.tuple IS NOT DISTINCT FROM bl.tuple
    AND kl.virtualxid IS NOT DISTINCT FROM bl.virtualxid
    AND kl.transactionid IS NOT DISTINCT FROM bl.transactionid
    AND kl.classid IS NOT DISTINCT FROM bl.classid
    AND kl.objid IS NOT DISTINCT FROM bl.objid
    AND kl.objsubid IS NOT DISTINCT FROM bl.objsubid
    AND kl.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE NOT bl.granted AND kl.granted;
```

### Lock Types Quick Reference

| Lock | Acquired by | Conflicts with |
|------|------------|----------------|
| `AccessShareLock` | SELECT | AccessExclusiveLock |
| `RowShareLock` | SELECT FOR UPDATE/SHARE | ExclusiveLock, AccessExclusiveLock |
| `RowExclusiveLock` | INSERT, UPDATE, DELETE | ShareLock, ShareRowExclusiveLock, ExclusiveLock, AccessExclusiveLock |
| `ShareLock` | CREATE INDEX (non-concurrent) | RowExclusiveLock and above |
| `AccessExclusiveLock` | ALTER TABLE, DROP TABLE, VACUUM FULL | Everything |

### Advisory Locks

For application-level coordination without row locking:

```sql
-- Choose one acquisition method, not both.
-- Blocking acquisition:
SELECT pg_advisory_lock(hashtext('my_job_name'));

-- Release once for each successful session-level acquisition:
SELECT pg_advisory_unlock(hashtext('my_job_name'));

-- Or use a non-blocking acquisition:
SELECT pg_try_advisory_lock(hashtext('my_job_name'));
SELECT pg_advisory_unlock(hashtext('my_job_name'));
```

## VACUUM and Bloat

### Check Autovacuum Status

```sql
SELECT
    schemaname,
    relname,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count,
    autoanalyze_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### VACUUM Time Tracking (PG18+)

PG18 adds cumulative timing columns to `pg_stat_all_tables`:

```sql
SELECT
    schemaname,
    relname,
    total_vacuum_time,
    total_autovacuum_time,
    total_analyze_time,
    total_autoanalyze_time
FROM pg_stat_user_tables
WHERE total_autovacuum_time > 0
ORDER BY total_autovacuum_time DESC
LIMIT 10;
```

### Autovacuum Tuning for High-Write Tables

```sql
-- Per-table autovacuum settings for a high-churn table
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- trigger at 1% dead tuples (default 20%)
    autovacuum_vacuum_cost_delay = 2,          -- faster vacuum (default 2ms)
    autovacuum_analyze_scale_factor = 0.005    -- trigger analyze more often
);

-- PG18+: fixed dead-tuple threshold (in addition to percentage-based)
-- autovacuum_vacuum_max_threshold = 100000000  -- trigger at fixed count regardless of table size
```

**PG18 vacuum improvements:**
- **Eager freezing**: Normal vacuums can freeze some pages opportunistically, reducing later freeze-only vacuum passes. Controlled by `vacuum_max_eager_freeze_failure_rate`.
- **`autovacuum_worker_slots`**: New GUC specifying max background worker slots; `autovacuum_max_workers` is now adjustable at runtime.
- **VACUUM/ANALYZE processes inheritance children by default**: Use `VACUUM (ONLY) tablename` for old behavior (parent table only).

### Manual VACUUM Operations

```sql
-- Standard VACUUM (reclaims space for reuse, doesn't lock)
VACUUM orders;

-- VACUUM with buffer usage limit (PG16+): limit shared buffer impact
VACUUM (BUFFER_USAGE_LIMIT '256kB') orders;

-- VACUUM with analysis
VACUUM ANALYZE orders;

-- VACUUM FULL (rewrites entire table — LOCKS TABLE, use as last resort)
VACUUM FULL orders;

-- VACUUM VERBOSE (show progress)
VACUUM VERBOSE orders;
```

### Monitoring VACUUM Progress

```sql
SELECT
    relid::regclass AS table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    CASE WHEN heap_blks_total > 0
         THEN round(100.0 * heap_blks_vacuumed / heap_blks_total, 1)
         ELSE 0 END AS pct_complete
FROM pg_stat_progress_vacuum;
```

## Connection Management

### Connection Overview

```sql
SELECT
    state,
    count(*) AS connections,
    max(now() - state_change) AS longest_in_state
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

### Idle Connections Holding Resources

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    now() - state_change AS idle_duration,
    left(query, 100) AS last_query
FROM pg_stat_activity
WHERE state = 'idle'
    AND now() - state_change > interval '10 minutes'
ORDER BY idle_duration DESC;
```

### Connection Limits

```sql
-- Current vs max connections
SELECT
    count(*) AS current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections;
```

If hitting connection limits, use PgBouncer or application-side connection pooling rather than increasing `max_connections`.

## pg_stat_statements

### Setup

```sql
-- In postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'

CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Top Queries by Total Time

```sql
SELECT
    left(query, 100) AS query,
    calls,
    round(total_exec_time::numeric, 1) AS total_ms,
    round(mean_exec_time::numeric, 1) AS avg_ms,
    round((100.0 * total_exec_time / nullif(sum(total_exec_time) OVER (), 0))::numeric, 1) AS pct_total,
    rows
FROM pg_stat_statements
WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database())
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Top Queries by Mean Time (Slowest Individual Executions)

```sql
SELECT
    left(query, 100) AS query,
    calls,
    round(mean_exec_time::numeric, 1) AS avg_ms,
    round(min_exec_time::numeric, 1) AS min_ms,
    round(max_exec_time::numeric, 1) AS max_ms,
    round(stddev_exec_time::numeric, 1) AS stddev_ms
FROM pg_stat_statements
WHERE calls >= 10  -- ignore rare queries
    AND dbid = (SELECT oid FROM pg_database WHERE datname = current_database())
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Parallel Worker Usage (PG18+)

PG18 adds parallel worker tracking to `pg_stat_statements`:

```sql
SELECT
    left(query, 80) AS query,
    calls,
    round(mean_exec_time::numeric, 1) AS avg_ms,
    parallel_workers_to_launch,
    parallel_workers_launched
FROM pg_stat_statements
WHERE parallel_workers_to_launch > 0
    AND dbid = (SELECT oid FROM pg_database WHERE datname = current_database())
ORDER BY parallel_workers_to_launch - parallel_workers_launched DESC
LIMIT 10;
```

A gap between `to_launch` and `launched` indicates insufficient parallel workers.

### Reset Statistics

```sql
-- Reset pg_stat_statements (do this after config changes or deployments)
SELECT pg_stat_statements_reset();

-- Reset table/index stats
SELECT pg_stat_reset();
```
