# Backup & Restore Reference

## Contents
- Backup strategy overview
- Logical backups (pg_dump / pg_restore)
- Physical backups (pg_basebackup)
- Point-in-Time Recovery (PITR)
- Verification and testing
- Automation patterns

## Backup Strategy Overview

| Method | What it captures | Granularity | Speed | Use case |
|--------|-----------------|-------------|-------|----------|
| `pg_dump` | Logical (SQL/custom) | Per-database, per-table | Slow on large DBs | Dev snapshots, migrations, selective restore |
| `pg_dumpall` | All databases + globals | Entire cluster | Slow | Full cluster backup including roles/tablespaces |
| `pg_basebackup` | Physical (file copy) | Entire cluster | Fast | Production backups, PITR base, replica setup |
| Continuous archiving | WAL segments | Incremental | Continuous | PITR — restore to any point in time |

**Production recommendation**: `pg_basebackup` + continuous WAL archiving for PITR capability. Supplement with periodic `pg_dump` for portable, version-independent backups.

## Logical Backups (pg_dump / pg_restore)

### pg_dump Formats

| Format | Flag | Parallel restore? | Selective restore? | Notes |
|--------|------|-------------------|-------------------|-------|
| Custom | `-Fc` | Yes | Yes | **Recommended default** — compressed, flexible |
| Directory | `-Fd` | Yes | Yes | One file per table, good for large DBs |
| Plain SQL | `-Fp` | No | No (manual editing) | Human-readable, good for version control |
| Tar | `-Ft` | No | Yes | Compatibility option |

### Common pg_dump Patterns

```bash
# Full database backup (custom format — recommended)
pg_dump -Fc -f backup.dump mydb

# With compression level (0-9, default varies by format)
pg_dump -Fc -Z 6 -f backup.dump mydb

# Parallel dump (directory format only, 4 workers)
pg_dump -Fd -j 4 -f backup_dir/ mydb

# Schema only (no data)
pg_dump -Fc --schema-only -f schema.dump mydb

# Data only (no schema)
pg_dump -Fc --data-only -f data.dump mydb

# Specific tables
pg_dump -Fc -t orders -t customers -f subset.dump mydb

# Specific schema
pg_dump -Fc -n public -f public_schema.dump mydb

# Exclude large tables
pg_dump -Fc -T audit_log -T event_archive -f without_logs.dump mydb

# Include CREATE DATABASE in output
pg_dump -Fc --create -f backup_with_create.dump mydb
```

### pg_restore Patterns

```bash
# Restore to an existing (empty) database
pg_restore -d mydb backup.dump

# Parallel restore (4 workers — significantly faster for large DBs)
pg_restore -d mydb -j 4 backup.dump

# Create the database during restore
pg_restore --create -d postgres backup.dump

# List contents of a backup (inspect before restoring)
pg_restore --list backup.dump

# Restore specific tables only
pg_restore -d mydb -t orders -t customers backup.dump

# Schema only
pg_restore -d mydb --schema-only backup.dump

# Data only (schema already exists)
pg_restore -d mydb --data-only backup.dump

# Clean (drop) objects before recreating
pg_restore -d mydb --clean --if-exists backup.dump

# Restore atomically and stop on the first error
pg_restore -d mydb --single-transaction -j 1 backup.dump
# Note: --single-transaction is incompatible with -j > 1
```

Selective restore patterns do not automatically include dependencies. Schema-qualify table patterns when names may exist in multiple schemas, and verify required types, sequences, constraints, and referenced tables separately.

### Restoring Plain SQL Dumps

```bash
# Plain SQL dumps are restored with psql, not pg_restore
psql -d mydb -f backup.sql

# With error handling
psql -d mydb -v ON_ERROR_STOP=1 -f backup.sql
```

### pg_dumpall — Cluster-Wide Backup

`pg_dumpall` is the only way to back up global objects (roles, tablespaces):

```bash
# Full cluster (all databases + globals) — plain SQL only
pg_dumpall -f cluster_backup.sql

# Globals only (roles, tablespaces) — use alongside per-database pg_dump
pg_dumpall --globals-only -f globals.sql

# Roles only
pg_dumpall --roles-only -f roles.sql
```

**Best practice**: Use `pg_dumpall --globals-only` for roles/tablespaces, then `pg_dump -Fc` per database for data. This gives you parallel restore capability while preserving globals.

### Optimizer Statistics in Backups (PG18+)

```bash
# Dump optimizer statistics (speeds up post-restore query performance)
pg_dump -Fc --statistics -f backup.dump mydb

# Statistics only (no schema or data)
pg_dump -Fc --statistics-only -f stats.dump mydb

# Skip statistics
pg_dump -Fc --no-statistics -f backup.dump mydb
```

Without statistics, the planner uses default estimates after restore until `ANALYZE` runs on all tables. Dumping statistics avoids the post-restore performance dip.

### Performance Tips for Large Databases

1. **Use parallel dump/restore** (`-Fd -j N`) — scales well with CPU cores
2. **Dump to fast storage** — local NVMe, not network mounts
3. **Increase `maintenance_work_mem`** on the restore target for faster index rebuilds
4. **Disable triggers during data-only restore**: `pg_restore --disable-triggers` (requires superuser)
5. **Drop indexes before restore, recreate after** — faster than incremental index maintenance during bulk inserts
6. **Restore schema first, then data, then indexes**:
   ```bash
   pg_restore -d mydb --section=pre-data backup.dump
   pg_restore -d mydb --data-only -j 4 backup.dump
   pg_restore -d mydb --section=post-data -j 4 backup.dump
   ```

## Physical Backups (pg_basebackup)

`pg_basebackup` creates a byte-for-byte copy of the entire cluster, suitable as a base for PITR or setting up replicas.

```bash
# Basic backup (plain format)
pg_basebackup -D /backup/base -Fp -Xs -P

# Compressed tar format
pg_basebackup -D /backup/base -Ft -z -Xs -P

# With checkpoint mode (fast = don't wait for next scheduled checkpoint)
pg_basebackup -D /backup/base -Fp -Xs -P --checkpoint=fast

# To a remote server
pg_basebackup -h primary_host -U repl_user -D /backup/base -Fp -Xs -P
```

| Flag | Meaning |
|------|---------|
| `-D` | Target directory |
| `-Fp` | Plain format (ready-to-use data directory) |
| `-Ft` | Tar format (one tar per tablespace) |
| `-z` | Compress (with tar format) |
| `-Xs` | Stream WAL during backup (ensures consistency) |
| `-P` | Show progress |
| `--checkpoint=fast` | Start backup immediately (don't wait for next checkpoint) |

### Prerequisites

The connection used by `pg_basebackup` must authenticate as a superuser or a role with `REPLICATION` privilege. `max_wal_senders` must also have enough capacity for the backup connection:

```sql
-- On the primary
CREATE ROLE backup_user WITH REPLICATION LOGIN PASSWORD '...';
```

And `pg_hba.conf` must allow replication connections:

```
host    replication     backup_user     backup_server_ip/32     scram-sha-256
```

[PG15+] `--target=server:/path` writes the backup on the database server. A non-superuser using this target needs both `REPLICATION` privilege for the backup connection and membership in `pg_write_server_files` for the server-side write. `--target` is unavailable in PG14 and cannot be combined with `-Xstream`; use `-Xfetch` or `-Xnone`.

## Point-in-Time Recovery (PITR)

PITR lets you restore a database to any specific point in time — invaluable for recovering from accidental data deletion or corruption.

### How PITR Works

1. Take a **base backup** (`pg_basebackup`)
2. Continuously **archive WAL segments** as they're produced
3. To recover: restore the base backup, then **replay WAL** up to the target time

### Step 1: Configure WAL Archiving

In `postgresql.conf`:

```
archive_mode = on
archive_command = 'cp %p /archive/wal/%f'    # or use pgBackRest, barman, etc.
# archive_library = ''  # PG15+: use archive modules instead of shell commands
```

Requires a restart after enabling `archive_mode`.

Verify archiving is working:

```sql
SELECT * FROM pg_stat_archiver;
-- Check: archived_count is increasing, last_failed_time is NULL
```

### Step 2: Take Base Backups Regularly

```bash
# Weekly base backup (adjust frequency based on WAL volume)
pg_basebackup -D /backup/base_$(date +%Y%m%d) -Ft -z -Xs -P --checkpoint=fast
```

### Step 3: Recovery

To recover to a specific point in time:

1. Stop PostgreSQL
2. Replace the data directory with the base backup
3. Create `recovery.signal` (PG12+) or `recovery.conf` (PG11-)
4. Configure recovery target in `postgresql.conf`:

```
# In postgresql.conf (PG12+):
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time = '2024-06-15 14:30:00'
recovery_target_action = 'promote'    # 'pause' to inspect before promoting
```

5. Start PostgreSQL — it replays WAL up to the target time, then promotes to read-write

### Recovery Target Options

```text
-- Recover to a specific time
recovery_target_time = '2024-06-15 14:30:00+00'

-- Recover to a specific transaction ID
recovery_target_xid = '12345678'

-- Recover to a named restore point
recovery_target_name = 'before_migration'

-- Stop as soon as a consistent state is reached
-- (for an online backup, normally the point where the backup ended)
recovery_target = 'immediate'

-- Recover to a specific WAL position
recovery_target_lsn = '0/1A2B3C4D'
```

To recover through the end of all available WAL, omit every `recovery_target*` setting. `recovery_target = 'immediate'` is an explicit early stopping target, so later available WAL can remain unapplied. PostgreSQL may still replay the WAL required to make an online backup consistent.

### Creating Named Restore Points

Before risky operations, create a named restore point:

```sql
SELECT pg_create_restore_point('before_schema_migration');
SELECT pg_create_restore_point('before_bulk_delete');
```

This gives you an exact target to recover to if the operation goes wrong.

## Verification and Testing

### Verify Backup Integrity

```bash
# List contents without restoring
pg_restore --list backup.dump

# Verify a plain-format physical backup (PG13+)
pg_verifybackup /backup/base_20240615

# PG18+: verify a tar-format backup directly
pg_verifybackup --no-parse-wal /backup/base_tar_20240615
```

### Test Restores Regularly

**A backup that hasn't been tested is not a backup.** Schedule regular restore tests:

```bash
# Restore to a test database
createdb mydb_restore_test
pg_restore -d mydb_restore_test backup.dump

# Verify row counts
psql -d mydb_restore_test -c "
SELECT schemaname, relname, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
"

# Clean up
dropdb mydb_restore_test
```

### Post-Restore Checklist

After restoring a backup:

1. **Run `ANALYZE`** on all tables — optimizer statistics may be stale or missing
   ```sql
   ANALYZE;  -- all tables
   ```
2. **Verify sequences** — if restoring data-only, sequences may not match the data
   ```sql
   SELECT sequencename, last_value FROM pg_sequences WHERE schemaname = 'public';
   ```
3. **Check for invalid indexes** — concurrent index builds may have been in progress
   ```sql
   SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid;
   ```
4. **Verify replication slots are clean** — stale slots from the source won't work
   ```sql
   SELECT slot_name, active FROM pg_replication_slots;
   ```

## Automation Patterns

### Scripted Backup with Retention

```bash
#!/bin/bash
BACKUP_DIR="/backup/pg"
RETENTION_DAYS=30
DB_NAME="mydb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Dump
pg_dump -Fc -Z 6 -f "${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.dump" "${DB_NAME}"

# Verify dump was created and is non-empty
if [ ! -s "${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.dump" ]; then
    echo "ERROR: Backup file is empty or missing" >&2
    exit 1
fi

# Clean old backups
find "${BACKUP_DIR}" -name "${DB_NAME}_*.dump" -mtime +${RETENTION_DAYS} -delete
```

### Dedicated Backup Tools

For production environments, consider purpose-built tools that handle scheduling, retention, compression, and PITR:

- **pgBackRest** — parallel backup/restore, incremental backups, S3/GCS/Azure support, built-in PITR
- **Barman** — backup management with retention policies, remote backup, WAL archiving
- **pg_probackup** — incremental backups with page-level tracking, merge, validation
