# Hot Standby & Read Replicas Reference

## Contents
- Streaming replication overview
- Hot standby configuration
- Monitoring replication lag
- Replication conflicts
- Synchronous vs asynchronous replication
- Promoting a standby

## Streaming Replication Overview

Physical (streaming) replication creates an exact copy of the primary database on one or more standby servers. Unlike logical replication, it replicates the entire cluster (all databases, all objects) at the WAL level.

**Hot standby** allows read-only queries on the standby while it continuously replays WAL from the primary. This is the standard mechanism for PostgreSQL read replicas.

### Key Differences from Logical Replication

| Aspect | Physical (streaming) | Logical |
|--------|---------------------|---------|
| Scope | Entire cluster | Selected tables |
| PG versions | Must match major version | Can differ |
| Standby writable? | No (read-only) | Yes (for non-replicated tables) |
| DDL replicated? | Yes (via WAL) | No |
| Use case | HA, read replicas | CDC, selective sync, cross-version migration |

## Hot Standby Configuration

### On the Primary

```sql
-- Check current settings
SHOW wal_level;              -- must be 'replica' or 'logical'
SHOW max_wal_senders;        -- must have available sender slots
SHOW max_replication_slots;  -- one per standby for slot-based replication
```

Key settings (in `postgresql.conf`):
- `wal_level = replica` (default)
- `max_wal_senders = 10` (enough for all standbys)
- `max_replication_slots = 10` (optional but recommended — prevents WAL removal before standby catches up)

### On the Standby

Key settings:
- `hot_standby = on` (allows read queries during recovery — default)
- `primary_conninfo = 'host=primary_host user=repl_user ...'` (connection to primary)
- `primary_slot_name = 'standby_slot'` (optional — use a replication slot)

### Replication Slots (Recommended)

Slots prevent the primary from removing WAL segments before the standby has replayed them:

```sql
-- On the primary: create a physical replication slot
SELECT pg_create_physical_replication_slot('standby1_slot');

-- List slots
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;

-- Drop a slot (if standby is permanently removed)
SELECT pg_drop_replication_slot('standby1_slot');
```

**Warning**: An inactive slot causes WAL to accumulate indefinitely on the primary, potentially filling the disk.

## Monitoring Replication Lag

### On the Primary: pg_stat_replication

```sql
SELECT
    pid,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) AS send_lag,
    write_lag,
    flush_lag,
    replay_lag AS replay_lag_time
FROM pg_stat_replication;
```

**LSN progression**: `pg_current_wal_lsn()` → `sent_lsn` → `write_lsn` → `flush_lsn` → `replay_lsn`. The gap between any two is a measure of lag at that stage.

### On the Standby: pg_stat_wal_receiver

```sql
SELECT
    status,
    written_lsn,
    flushed_lsn,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    conninfo
FROM pg_stat_wal_receiver;
```

### On the Standby: Lag in Seconds

```sql
-- How far behind is the standby?
SELECT
    now() - pg_last_xact_replay_timestamp() AS replay_lag,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_lag_bytes;
```

**Caveat**: `pg_last_xact_replay_timestamp()` only updates when transactions are replayed. On an idle primary, it may show a large lag even though the standby is fully caught up. Check `received_lsn = replayed_lsn` for actual status.

### Is the Standby in Recovery?

```sql
-- Returns true on a standby, false on a primary
SELECT pg_is_in_recovery();
```

## Replication Conflicts

On a hot standby, long-running read queries can conflict with WAL replay. When replay needs to apply changes that conflict with an active query (e.g., dropping a table, vacuuming rows the query is reading), PostgreSQL must choose: wait for the query or cancel it.

### max_standby_streaming_delay

Controls how long the standby waits for conflicting queries before cancelling them:

```sql
SHOW max_standby_streaming_delay;  -- default: 30s
```

- `30s` (default): Standby waits up to 30 seconds, then cancels conflicting queries
- `-1`: Wait forever (replay pauses until the query finishes — can cause unbounded lag)
- `0`: Cancel conflicting queries immediately (minimal lag, but queries may fail)

### max_standby_archive_delay

Same concept but for WAL segments being replayed from archive (rather than streaming):

```sql
SHOW max_standby_archive_delay;  -- default: 30s
```

### The Common Error

When a query on the standby is cancelled due to a conflict:

```
ERROR: canceling statement due to conflict with recovery
DETAIL: User was holding shared buffer pin for too long.
```

**Solutions (in order of preference):**
1. Keep queries on the standby short
2. Increase `max_standby_streaming_delay` (trades lag for query stability)
3. Enable `hot_standby_feedback` (see below)
4. Use a logical replica instead of hot standby for long-running analytics

### hot_standby_feedback

When enabled, the standby informs the primary about which rows it still needs, preventing the primary's VACUUM from removing them:

```sql
SHOW hot_standby_feedback;  -- default: off
```

**Pros**: Eliminates most replication conflicts — long queries on the standby won't be cancelled.

**Cons**: Can cause table bloat on the primary because VACUUM can't remove dead rows that the standby still references.

**Recommendation**: Enable only if you have long-running analytical queries on the standby and can tolerate some extra bloat on the primary.

## Synchronous vs Asynchronous Replication

### Asynchronous (Default)

The primary doesn't wait for standby acknowledgment before committing. Fastest, but a primary failure can lose recently committed transactions not yet replicated.

### Synchronous

The primary waits for at least one standby to confirm before returning commit success:

```sql
-- On the primary
SHOW synchronous_standby_names;  -- e.g., 'standby1'
SHOW synchronous_commit;         -- 'on', 'remote_write', 'remote_apply', etc.
```

Synchronous commit levels:

| Level | Primary waits for | Durability | Latency impact |
|-------|-------------------|------------|----------------|
| `on` (default with sync standbys) | Standby WAL flush | Strong | Moderate |
| `remote_write` | Standby WAL write (not fsync) | Good | Lower |
| `remote_apply` | Standby WAL replay | Strongest (read-your-writes on standby) | Highest |
| `local` | Local WAL flush only | Primary only | None |
| `off` | Nothing | Weakest | None |

## Promoting a Standby

To promote a standby to become the new primary (failover):

```sql
-- On the standby:
SELECT pg_promote();

-- Or from the command line:
-- pg_ctl promote -D /path/to/data
```

After promotion:
- The standby stops replay and opens for writes
- Applications must be redirected to the new primary
- Other standbys must be reconfigured to follow the new primary
- The old primary must not be restarted without reconfiguring (risk of split-brain)
