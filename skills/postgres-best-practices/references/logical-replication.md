# Logical Replication & Migrations Reference

## Contents
- Overview and prerequisites
- Publisher setup
- Subscriber setup
- Managing publications (add/remove tables)
- Managing subscriptions
- Monitoring replication progress
- Schema changes during replication
- Live migration patterns
- Troubleshooting

## Overview

Logical replication streams row-level changes (INSERT, UPDATE, DELETE) from a **publisher** to one or more **subscribers** using the publish/subscribe model. Unlike physical replication, it:

- Replicates specific tables, not the entire cluster
- Allows different indexes, security policies, or schemas on the subscriber
- Works across different PG major versions (useful for upgrades)
- Allows the subscriber to be writable (for other tables)

### Prerequisites

**On the publisher:**
- `wal_level = logical` (requires restart if changing)
- Sufficient `max_replication_slots` (one per subscription)
- Sufficient `max_wal_senders` (one per subscription + headroom)
- Tables must have a replica identity (primary key by default)
- The replication role needs `REPLICATION` privilege, plus `USAGE` on the schema and `SELECT` on replicated tables

**On the subscriber:**
- Target tables must already exist with compatible schema
- Sufficient `max_logical_replication_workers`
- Sufficient `max_worker_processes`
- The role needs `pg_create_subscription` membership (PG16+)

Check current settings:

```sql
SHOW wal_level;               -- must be 'logical'
SHOW max_replication_slots;   -- must have available slots
SHOW max_wal_senders;         -- must have available senders

-- Check how many slots are already in use
SELECT count(*) AS used_slots FROM pg_replication_slots;
```

**How many slots does a subscription need?**

During **initial sync**, Postgres creates one temporary replication slot per table being copied in parallel, plus one permanent slot for the subscription itself. For example, a subscription syncing 10 tables with `max_sync_workers_per_subscription = 2` (default) uses up to **3 slots** at peak: 1 permanent + 2 temporary for parallel table copy.

Once initial sync completes and the subscription enters **streaming (CDC) mode**, only the **1 permanent slot** per subscription is used. The temporary per-table slots are dropped.

**Rule of thumb**: `max_replication_slots` >= (number of subscriptions) + (max_sync_workers_per_subscription) + headroom for physical replication slots. The default of 10 is sufficient for most setups.

## Publisher Setup

### Create a Publication

```sql
-- Publish specific tables
CREATE PUBLICATION my_pub FOR TABLE orders, customers, products;

-- Publish all tables in a schema (PG15+)
CREATE PUBLICATION my_pub FOR TABLES IN SCHEMA public;

-- Publish all tables in the database
-- WARNING: FOR ALL TABLES prevents later ADD/DROP TABLE modifications.
-- Prefer listing tables explicitly if you may need to change the set later.
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- Publish only specific operations
CREATE PUBLICATION inserts_only FOR TABLE events
    WITH (publish = 'insert');

-- Publish with row filter (PG15+)
CREATE PUBLICATION active_orders FOR TABLE orders
    WHERE (status = 'active');

-- Publish specific columns only (PG15+)
CREATE PUBLICATION partial_customers FOR TABLE customers (id, name, email);
```

### Replica Identity

Logical replication needs a way to identify rows for UPDATE and DELETE. By default, it uses the primary key.

```sql
-- Check current replica identity
SELECT relname, relreplident
FROM pg_class
WHERE relname IN ('orders', 'customers');
-- 'd' = default (primary key), 'f' = full, 'n' = nothing, 'i' = index

-- If a table has no primary key, use FULL (sends entire old row)
ALTER TABLE legacy_table REPLICA IDENTITY FULL;

-- Or use a unique index
CREATE UNIQUE INDEX idx_legacy_key ON legacy_table(external_id);
ALTER TABLE legacy_table REPLICA IDENTITY USING INDEX idx_legacy_key;
```

**Without a replica identity, UPDATE and DELETE will fail on the publisher** for that table.

### Grant Permissions to the Replication Role

The replication role needs schema access and SELECT on the published tables:

```sql
GRANT USAGE ON SCHEMA public TO repl_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO repl_user;

-- Also grant for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO repl_user;
```

### Decoder Plugins

Postgres supports logical decoding output plugins, including:
- **`pgoutput`** (default): built into Postgres, used by native logical replication
- **`wal2json`**: an optional third-party plugin that converts WAL to JSON format for CDC integrations

The decoder is specified when creating a replication slot manually:

```sql
SELECT pg_create_logical_replication_slot('my_slot', 'pgoutput');
```

After installing `wal2json` on the database server:

```sql
SELECT pg_create_logical_replication_slot('my_slot', 'wal2json');
```

When using `CREATE SUBSCRIPTION`, the default `pgoutput` plugin is used automatically.

### List Publications

```sql
SELECT * FROM pg_publication;

-- See which tables are in a publication
SELECT pubname, schemaname, tablename
FROM pg_publication_tables
ORDER BY pubname, tablename;
```

## Subscriber Setup

### Create a Subscription

```sql
-- Basic subscription (triggers initial data copy)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher_host dbname=source_db user=repl_user password=...'
    PUBLICATION my_pub;

-- Subscribe without initial data copy (tables already have data)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher_host dbname=source_db user=repl_user password=...'
    PUBLICATION my_pub
    WITH (copy_data = false);

-- Subscribe to multiple publications
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher_host dbname=source_db user=repl_user password=...'
    PUBLICATION pub_orders, pub_customers;

-- Create disabled (activate later)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher_host dbname=source_db user=repl_user password=...'
    PUBLICATION my_pub
    WITH (enabled = false);
```

### List Subscriptions

```sql
SELECT * FROM pg_subscription;

-- Subscription status and worker info
SELECT subname, pid, relid::regclass, received_lsn, latest_end_lsn,
       latest_end_time
FROM pg_stat_subscription;
```

## Managing Publications

### Add Tables

```sql
ALTER PUBLICATION my_pub ADD TABLE new_table;

-- Add with row filter (PG15+)
ALTER PUBLICATION my_pub ADD TABLE audit_log WHERE (created_at > '2024-01-01');

-- Add multiple tables at once
ALTER PUBLICATION my_pub ADD TABLE table_a, table_b, table_c;
```

### Remove Tables

```sql
ALTER PUBLICATION my_pub DROP TABLE old_table;

-- Remove multiple
ALTER PUBLICATION my_pub DROP TABLE table_a, table_b;
```

### Replace Entire Table List

```sql
ALTER PUBLICATION my_pub SET TABLE orders, customers, products, shipments;
```

### Change Published Operations

```sql
-- Only publish inserts and updates (no deletes)
ALTER PUBLICATION my_pub SET (publish = 'insert, update');

-- Restore all operations
ALTER PUBLICATION my_pub SET (publish = 'insert, update, delete, truncate');
```

### After Adding Tables — Refresh the Subscriber

After adding tables to a publication, the subscriber must refresh:

```sql
-- On the subscriber: pick up new tables and copy initial data
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;

-- Refresh without copying data for new tables
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION WITH (copy_data = false);
```

### Drop a Publication

```sql
DROP PUBLICATION my_pub;
-- or
DROP PUBLICATION IF EXISTS my_pub;
```

## Managing Subscriptions

### Enable / Disable

```sql
-- Pause replication
ALTER SUBSCRIPTION my_sub DISABLE;

-- Resume replication
ALTER SUBSCRIPTION my_sub ENABLE;
```

### Change Connection

```sql
ALTER SUBSCRIPTION my_sub CONNECTION 'host=new_host dbname=source_db user=repl_user password=...';
```

### Change Publications

```sql
-- Switch to different publications
ALTER SUBSCRIPTION my_sub SET PUBLICATION new_pub;

-- Add a publication
ALTER SUBSCRIPTION my_sub ADD PUBLICATION extra_pub;

-- Remove a publication
ALTER SUBSCRIPTION my_sub DROP PUBLICATION old_pub;
```

### Drop a Subscription

```sql
-- This also drops the replication slot on the publisher
DROP SUBSCRIPTION my_sub;
```

If the publisher is unreachable, disable first then drop:

```sql
ALTER SUBSCRIPTION my_sub DISABLE;
ALTER SUBSCRIPTION my_sub SET (slot_name = NONE);
DROP SUBSCRIPTION my_sub;
-- Then manually drop the orphaned slot on the publisher when it's back:
-- SELECT pg_drop_replication_slot('my_sub');
```

## Monitoring Replication Progress

### On the Publisher: Replication Slots and Lag

```sql
-- Replication slot status
SELECT
    slot_name,
    plugin,
    slot_type,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS replication_lag
FROM pg_replication_slots
WHERE slot_type = 'logical';

-- Active WAL senders
SELECT
    pid,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag
FROM pg_stat_replication;
```

### On the Subscriber: Subscription Status

```sql
-- Overall subscription status
SELECT
    subname,
    pid,
    received_lsn,
    latest_end_lsn,
    latest_end_time,
    last_msg_send_time,
    last_msg_receipt_time
FROM pg_stat_subscription
WHERE subname IS NOT NULL;

-- Per-table sync state (initial copy progress)
SELECT
    s.subname AS subscription,
    sr.srrelid::regclass AS table_name,
    sr.srsubstate AS state,
    CASE sr.srsubstate
        WHEN 'i' THEN 'initialize'
        WHEN 'd' THEN 'data is being copied'
        WHEN 'f' THEN 'finished table copy'
        WHEN 's' THEN 'synchronized'
        WHEN 'r' THEN 'ready (normal replication)'
        ELSE 'unknown'
    END AS state_meaning,
    sr.srsublsn AS lsn
FROM pg_catalog.pg_subscription_rel AS sr
JOIN pg_catalog.pg_subscription AS s
  ON s.oid = sr.srsubid
ORDER BY s.subname, sr.srrelid::regclass::text;
```

**State codes for `pg_subscription_rel.srsubstate`:**

| Code | Meaning |
|------|---------|
| `i` | Initializing |
| `d` | Copying data (initial sync) |
| `f` | Finished table copy, waiting for sync |
| `s` | Synced with publisher |
| `r` | Ready (streaming) |

These codes describe each table's initialization state. `s` means the table synchronized during initialization; it does not prove that current replication lag is zero.

### Replication Lag Monitoring Query

Run on the subscriber to check how far behind it is:

```sql
-- Lag in bytes and time
SELECT
    s.subname,
    st.received_lsn,
    st.latest_end_lsn,
    st.latest_end_time,
    now() - st.latest_end_time AS time_lag
FROM pg_subscription s
JOIN pg_stat_subscription st ON st.subid = s.oid
WHERE st.pid IS NOT NULL
    AND st.relid IS NULL
    -- PG17+ can also expose parallel apply workers; keep only the leader.
    AND coalesce(to_jsonb(st)->>'worker_type', 'apply') = 'apply';
```

## Schema Changes During Replication

Logical replication does **NOT** replicate DDL. Schema changes must be applied manually on both sides.

### Safe Pattern for Adding a Column

```sql
-- 1. Add column on SUBSCRIBER first (nullable, no default)
ALTER TABLE orders ADD COLUMN priority int;

-- 2. Add column on PUBLISHER
ALTER TABLE orders ADD COLUMN priority int;

-- 3. New rows will now include the column
-- Existing replicated rows will have NULL for the new column
```

**Add on subscriber first** to avoid errors when the publisher starts sending the new column before the subscriber schema is updated.

### Safe Pattern for Dropping a Column

```sql
-- 1. Remove column from PUBLICATION (PG15+, if using column lists)
ALTER PUBLICATION my_pub SET TABLE orders (id, customer_id, total, created_at);

-- 2. Drop column on PUBLISHER
ALTER TABLE orders DROP COLUMN old_column;

-- 3. Drop column on SUBSCRIBER
ALTER TABLE orders DROP COLUMN old_column;
```

### Adding a New Table to Replication

```sql
-- 1. Create table on SUBSCRIBER with matching schema
CREATE TABLE shipments (...);

-- 2. Add table to publication on PUBLISHER
ALTER PUBLICATION my_pub ADD TABLE shipments;

-- 3. Refresh on SUBSCRIBER (copies existing data)
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;
```

## Live Migration Patterns

### Migration to a New Database (Minimal Downtime)

1. **Set up target**: Create schema and constraints on the new database. **Defer index creation** until after initial data copy completes — this significantly speeds up the initial sync.
2. **Start replication**: Create publication on source, subscription on target
3. **Wait for sync**: Monitor until all tables reach `r` (ready) state
4. **Verify**: Compare row counts, spot-check data
5. **Cutover**:
   - Stop writes to source (set `default_transaction_read_only = on` or revoke write access)
   - Wait for final lag to drain to zero
   - Verify sequences: advance sequences on target to match source
   - Switch application connection strings
6. **Cleanup**: Drop subscription, drop publication, drop replication slot

### Sequence Synchronization

Logical replication does **NOT** replicate sequences. Before cutover, sync them:

```sql
-- On SOURCE: get current sequence values
SELECT schemaname, sequencename, last_value
FROM pg_sequences
WHERE schemaname = 'public';

-- On TARGET: substitute the value obtained from the source.
-- Add a buffer only if writes can still reach the source during cutover.
SELECT pg_catalog.setval(
    'public.orders_id_seq'::regclass,
    <source_last_value> + 1000,
    true
);
```

### Row Count Verification

```sql
-- Run on both source and target, compare results
SELECT
    schemaname,
    relname,
    n_live_tup AS approx_row_count
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY relname;
```

For exact counts (slower):

```sql
CREATE TEMP TABLE exact_row_counts (
    schemaname name NOT NULL,
    tablename name NOT NULL,
    exact_count bigint NOT NULL
);

DO $$
DECLARE r record;
BEGIN
    FOR r IN
        SELECT schemaname, tablename
        FROM pg_catalog.pg_tables
        WHERE schemaname = 'public'
        ORDER BY tablename
    LOOP
        EXECUTE format(
            'INSERT INTO pg_temp.exact_row_counts
             SELECT %L::name, %L::name, count(*)
             FROM %I.%I',
            r.schemaname, r.tablename,
            r.schemaname, r.tablename
        );
    END LOOP;
END $$;

SELECT
    format('%I.%I', schemaname, tablename) AS table_name,
    exact_count
FROM pg_temp.exact_row_counts
ORDER BY schemaname, tablename;

DROP TABLE pg_temp.exact_row_counts;
```

Run the exact-count query on both source and target only after stopping writes and allowing replication lag to drain. It performs a full scan of every selected table.

## Troubleshooting

### Replication Slot Growing / WAL Accumulation

If a subscriber falls behind or is disconnected, WAL accumulates on the publisher:

```sql
-- Check slot lag on publisher
SELECT slot_name, active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

**Fix**: Reconnect the subscriber, or drop the slot if the subscription is no longer needed:

```sql
-- Drop an inactive slot (publisher)
SELECT pg_drop_replication_slot('orphaned_slot_name');
```

### Initial Sync Stuck or Slow

```sql
-- Check which tables are still copying
SELECT srrelid::regclass AS table_name, srsubstate AS state
FROM pg_subscription_rel
WHERE srsubstate != 'r';
-- 'd' = still copying data, 'i' = initializing
```

Initial sync speed depends on table size and network. For very large tables, consider:
- Dump/restore the data first, then create subscription with `copy_data = false`
- Increase `max_logical_replication_workers` if many tables need initial sync in parallel

### Conflict Errors

If the subscriber has conflicting data (e.g., duplicate key):

```sql
-- PG15+: cumulative subscription apply-error counters
SELECT * FROM pg_stat_subscription_stats;
```

This view reports counters rather than detailed error messages. Check PostgreSQL server logs for the specific conflict; on PG14, logs are the primary source because `pg_stat_subscription_stats` is unavailable.

**PG16+**: Skip a conflicting transaction:

```sql
ALTER SUBSCRIPTION my_sub SKIP (lsn = '0/12345678');
```

**Pre-PG16**: Delete the conflicting row on the subscriber, then replication will proceed.

### "Publisher Does Not Exist" After Refresh

If you renamed or recreated a publication:

```sql
-- Subscriber needs to be pointed to the new publication name
ALTER SUBSCRIPTION my_sub SET PUBLICATION new_pub_name;
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;
```

### Inactive Slot Timeout (PG18+)

PG18 adds `idle_replication_slot_timeout` to automatically invalidate inactive replication slots, preventing unbounded WAL accumulation:

```sql
SHOW idle_replication_slot_timeout;  -- auto-invalidates stale slots
```

### Generated Column Replication (PG18+)

Logical replication can now replicate generated column values via the `publish_generated_columns` publication option:

```sql
CREATE PUBLICATION my_pub FOR TABLE orders
    WITH (publish_generated_columns = true);
```

### Check if wal_level is Logical

```sql
SHOW wal_level;
-- If not 'logical', it requires a restart to change
-- In postgresql.conf: wal_level = logical
```

On managed platforms, `wal_level` may be controlled by the platform's settings UI or API rather than `postgresql.conf`.
