# Bulk Data Loading Reference

## Contents
- COPY vs INSERT performance
- COPY FROM patterns
- Optimizing large loads
- ETL staging patterns
- Bulk updates and deletes

## COPY vs INSERT Performance

| Method | Rows/sec (typical) | Use case |
|--------|-------------------|----------|
| Single-row INSERT | ~1,000-5,000 | Application writes |
| Multi-row INSERT (VALUES) | ~10,000-50,000 | Batch inserts from code |
| `COPY FROM` | ~100,000-500,000+ | Bulk loading from files or streams |
| `COPY FROM` (binary) | ~200,000-1,000,000+ | Maximum throughput (binary format) |

`COPY` is **10-100x faster** than individual INSERTs because it bypasses per-row overhead (parsing, planning, WAL per statement).

## COPY FROM Patterns

### From a File

```sql
-- CSV with header
COPY orders FROM '/path/to/orders.csv' WITH (FORMAT csv, HEADER true);

-- Tab-delimited
COPY orders FROM '/path/to/orders.tsv' WITH (FORMAT text);

-- Custom delimiter
COPY orders FROM '/path/to/orders.dat' WITH (DELIMITER '|');

-- With NULL handling
COPY orders FROM '/path/to/orders.csv'
    WITH (FORMAT csv, HEADER true, NULL '');
```

### From STDIN (Piped Data)

```bash
# Pipe from another command
cat orders.csv | psql -d mydb -c "\COPY orders FROM STDIN WITH (FORMAT csv, HEADER true)"

# Pipe from gzip
gunzip -c orders.csv.gz | psql -d mydb -c "\COPY orders FROM STDIN WITH (FORMAT csv, HEADER true)"
```

### From Application Code

Most drivers support COPY protocol for streaming data:

```python
# Python (psycopg 3)
with conn.cursor() as cur:
    with cur.copy("COPY orders (id, customer_id, total) FROM STDIN") as copy:
        for row in data:
            copy.write_row(row)
```

```javascript
// Node.js (pg-copy-streams)
const { from } = require('pg-copy-streams');
const stream = client.query(from('COPY orders FROM STDIN WITH (FORMAT csv)'));
fileStream.pipe(stream);
```

### COPY TO (Export)

```sql
-- Export to CSV
COPY orders TO '/path/to/orders.csv' WITH (FORMAT csv, HEADER true);

-- Export query results
COPY (SELECT id, total FROM orders WHERE created_at > '2024-01-01')
    TO '/path/to/recent.csv' WITH (FORMAT csv, HEADER true);
```

### Error Handling (PG17+)

```sql
-- Skip invalid rows instead of failing the entire COPY
COPY orders FROM '/path/to/orders.csv'
    WITH (FORMAT csv, HEADER true, ON_ERROR ignore);

-- PG18+: limit how many errors to tolerate
COPY orders FROM '/path/to/orders.csv'
    WITH (FORMAT csv, HEADER true, ON_ERROR ignore, REJECT_LIMIT 100);
```

## Optimizing Large Loads

### 1. Drop Indexes, Load, Recreate

Building indexes incrementally during COPY is much slower than building them once after:

```sql
-- Before load: drop non-essential indexes
DROP INDEX idx_orders_customer_id;
DROP INDEX idx_orders_created_at;

-- Load data
COPY orders FROM '/path/to/orders.csv' WITH (FORMAT csv, HEADER true);

-- After load: recreate indexes (concurrently if table is in use)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Update statistics
ANALYZE orders;
```

### 2. Disable Triggers During Load

```sql
-- Disable all triggers on the table
ALTER TABLE orders DISABLE TRIGGER ALL;

-- Load data
COPY orders FROM '/path/to/orders.csv' WITH (FORMAT csv, HEADER true);

-- Re-enable triggers
ALTER TABLE orders ENABLE TRIGGER ALL;
```

`DISABLE TRIGGER ALL` requires superuser privileges when the table has foreign-key or other internally generated constraint triggers. A table owner can use `DISABLE TRIGGER USER` to disable only user-defined triggers. Disabling constraint triggers can admit invalid data, so validate constraints before re-enabling writes.

### 3. Increase maintenance_work_mem

Larger `maintenance_work_mem` speeds up index creation after the load:

```sql
SET maintenance_work_mem = '1GB';  -- for the duration of the load session
```

### 4. Disable Autovacuum During Load

For very large bulk loads, temporarily disable autovacuum to avoid competing I/O:

```sql
ALTER TABLE orders SET (autovacuum_enabled = false);

-- Load data...

ALTER TABLE orders SET (autovacuum_enabled = true);
VACUUM ANALYZE orders;
```

### 5. Use Unlogged Tables for Staging

Unlogged tables skip WAL writes — 2-3x faster for writes but **data is lost on crash**:

```sql
CREATE UNLOGGED TABLE staging_orders (LIKE orders INCLUDING ALL);

COPY staging_orders FROM '/path/to/orders.csv' WITH (FORMAT csv, HEADER true);

-- Transform and move to the real table
INSERT INTO orders SELECT * FROM staging_orders;

DROP TABLE staging_orders;
```

### 6. Batch Size for Programmatic Inserts

When COPY isn't available, use multi-row INSERT with batches of 100-1,000 rows:

```sql
-- Single round-trip for 1,000 rows
INSERT INTO orders (customer_id, total, created_at)
VALUES
    (1, 99.99, now()),
    (2, 149.99, now()),
    -- ... up to ~1,000 rows per statement
    (1000, 79.99, now());
```

Beyond ~1,000 rows per statement, parse overhead increases. Use COPY for larger batches.

### 7. Parallel Loading into Partitioned Tables

Load data into individual partitions concurrently from separate sessions:

```bash
# Session 1
psql -c "COPY events_2024_q1 FROM '/data/q1.csv' WITH (FORMAT csv)"

# Session 2 (concurrent)
psql -c "COPY events_2024_q2 FROM '/data/q2.csv' WITH (FORMAT csv)"
```

## ETL Staging Patterns

### Staging Table with Upsert

```sql
-- Create staging table (temporary or unlogged)
CREATE TEMP TABLE staging_customers (LIKE customers INCLUDING DEFAULTS);

-- Load raw data
COPY staging_customers FROM '/path/to/customers.csv' WITH (FORMAT csv, HEADER true);

-- Upsert into production table
INSERT INTO customers (id, name, email, updated_at)
SELECT id, name, email, now()
FROM staging_customers
ON CONFLICT (id)
DO UPDATE SET
    name = EXCLUDED.name,
    email = EXCLUDED.email,
    updated_at = EXCLUDED.updated_at;
```

### Staging Table with MERGE (PG15+)

```sql
MERGE INTO customers AS target
USING staging_customers AS source
ON target.id = source.id
WHEN MATCHED AND source.name IS DISTINCT FROM target.name THEN
    UPDATE SET name = source.name, email = source.email, updated_at = now()
WHEN NOT MATCHED THEN
    INSERT (id, name, email, updated_at)
    VALUES (source.id, source.name, source.email, now());
```

### Swap Table Pattern

For full-refresh loads where you replace all data:

```sql
-- Load into a new table
CREATE TABLE orders_new (LIKE orders INCLUDING ALL);
COPY orders_new FROM '/path/to/orders.csv' WITH (FORMAT csv, HEADER true);
ANALYZE orders_new;

-- Atomic swap (brief exclusive lock)
BEGIN;
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_new RENAME TO orders;
DROP TABLE orders_old;
COMMIT;
```

`LIKE ... INCLUDING ALL` does not copy foreign keys, triggers, rules, grants, row-level security policies, or publication membership. Recreate and verify those objects before swapping, or use a data-only refresh when the original table's identity and dependencies must remain unchanged.

## Bulk Updates and Deletes

### Chunked Deletes

Large DELETEs lock rows and generate WAL. This loop limits each statement to 10,000 rows:

```sql
-- Delete in batches of 10,000
DO $$
DECLARE
    rows_deleted int;
BEGIN
    LOOP
        DELETE FROM audit_log
        WHERE id IN (
            SELECT id FROM audit_log
            WHERE created_at < now() - interval '1 year'
            LIMIT 10000
        );
        GET DIAGNOSTICS rows_deleted = ROW_COUNT;
        EXIT WHEN rows_deleted = 0;
        -- Optional: brief pause to reduce WAL pressure
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

The entire `DO` block is still one transaction: locks, WAL, and dead rows accumulate until it finishes. For true transaction-level batching, execute one limited DELETE per client transaction and commit between batches.

### Chunked Updates

The same statement-size pattern works for large updates:

```sql
DO $$
DECLARE
    rows_updated int;
BEGIN
    LOOP
        UPDATE orders
        SET status = 'archived'
        WHERE id IN (
            SELECT id FROM orders
            WHERE status = 'completed'
            AND created_at < now() - interval '2 years'
            LIMIT 10000
            FOR UPDATE SKIP LOCKED
        );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
    END LOOP;
END $$;
```

As with the DELETE loop, the `DO` block commits only once. Drive batches from the client or a transaction-controlling procedure invoked outside an explicit transaction when each batch must commit independently.

### Partition Drop Instead of Delete

If data is partitioned by time, dropping a partition is instant vs. a slow DELETE:

```sql
-- Instant: drop old partition
ALTER TABLE events DETACH PARTITION events_2023_q1 CONCURRENTLY;
DROP TABLE events_2023_q1;

-- vs. slow: delete rows
-- DELETE FROM events WHERE occurred_at < '2024-01-01';  -- don't do this
```
