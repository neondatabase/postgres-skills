# Connection Pooling Reference

## Contents
- Why connection pooling matters
- PgBouncer configuration
- Pool modes
- Prepared statement handling
- Monitoring and diagnostics
- Application-side pooling

## Why Connection Pooling Matters

Each PostgreSQL connection spawns a dedicated backend process (~5-10 MB of memory). Without pooling:

- 500 application instances × 10 connections each = 5,000 backend processes
- Memory: 5,000 × 10 MB = ~50 GB just for connection overhead
- Context switching degrades performance above a few hundred active backends
- `max_connections` must be set high, wasting shared memory

A connection pooler sits between the application and PostgreSQL, multiplexing many client connections onto fewer server connections.

### Connection Overhead

```sql
-- Current connections vs. limit
SELECT
    count(*) AS current,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max,
    (SELECT setting::int FROM pg_settings WHERE name = 'superuser_reserved_connections') AS reserved
FROM pg_stat_activity;

-- Potential per-operation and per-session memory settings
SELECT
    name,
    current_setting(name) AS configured_value,
    pg_size_bytes(current_setting(name)) AS configured_bytes
FROM pg_settings
WHERE name IN ('work_mem', 'temp_buffers')
ORDER BY name;
```

These are not baseline allocations per backend. `work_mem` can be consumed by multiple query operations, while `temp_buffers` is allocated lazily when a session uses temporary tables.

## PgBouncer Configuration

PgBouncer is the most widely used connection pooler for PostgreSQL.

### Essential pgbouncer.ini Settings

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool sizing
pool_mode = transaction
default_pool_size = 20
min_pool_size = 5
max_client_conn = 1000
max_db_connections = 50

# Timeouts
server_idle_timeout = 300
client_idle_timeout = 0
query_timeout = 0
query_wait_timeout = 120
server_login_retry = 15

# Logging
log_connections = 0
log_disconnections = 0
stats_period = 60
```

### Key Parameters

| Parameter | Recommended | Purpose |
|-----------|------------|---------|
| `default_pool_size` | 20-50 | Server connections per user/database pair |
| `min_pool_size` | 5 | Minimum idle server connections to keep open |
| `max_client_conn` | 1000-10000 | Max client connections PgBouncer accepts |
| `max_db_connections` | 50-100 | Hard cap on server connections per database |
| `reserve_pool_size` | 5 | Extra connections for burst traffic |
| `reserve_pool_timeout` | 3 | Seconds before using reserve pool |
| `query_wait_timeout` | 120 | Max time a client waits for a server connection |

### Sizing Rule of Thumb

**Server connections**: Set `default_pool_size` to roughly 2-4x the number of CPU cores on the database server. More connections don't help — they just increase lock contention and context switching.

**Client connections**: Set `max_client_conn` high enough for all application instances. PgBouncer handles thousands of idle client connections with minimal memory.

```
App instances (500 clients) ──→ PgBouncer (max_client_conn=1000) ──→ PostgreSQL (pool_size=20)
```

## Pool Modes

### Transaction Mode (Recommended for Most Applications)

```ini
pool_mode = transaction
```

Server connection is assigned when a transaction begins and returned when it commits/rolls back. Between transactions, the connection is available to other clients.

**Compatible with**: Standard SQL, parameterized queries, most ORMs.

**NOT compatible with** (session-level features):
- `SET` / `RESET` (use `SET LOCAL` inside a transaction instead)
- `LISTEN` / `NOTIFY`
- SQL-level `PREPARE` / `DEALLOCATE` (use protocol-level prepared statements)
- `DECLARE ... WITH HOLD` cursors
- Temporary tables with `ON COMMIT PRESERVE ROWS`
- Session-level advisory locks (`pg_advisory_lock` — use `pg_advisory_xact_lock` instead)
- `LOAD` statement

### Session Mode

```ini
pool_mode = session
```

Server connection is held for the entire client session. Compatible with all PostgreSQL features but offers less multiplexing benefit.

**Use when**: Application relies on session-level features (temp tables, LISTEN/NOTIFY, SET parameters).

### Statement Mode

```ini
pool_mode = statement
```

Server connection released after every statement. Maximum multiplexing but **incompatible with multi-statement transactions**.

**Use when**: Application only runs autocommit single statements (rare).

### Choosing a Mode

| Application pattern | Recommended mode |
|--------------------|-----------------|
| Web apps, APIs, serverless | Transaction |
| Applications using LISTEN/NOTIFY | Session |
| Applications using temp tables across transactions | Session |
| Applications with SET session variables | Session (or refactor to `SET LOCAL`) |
| Simple autocommit queries | Statement |

## Prepared Statement Handling

### The Problem

SQL-level `PREPARE`/`EXECUTE` creates server-side prepared statements tied to a session. In transaction mode, the next transaction may use a different server connection where the prepared statement doesn't exist.

### Solutions

**Protocol-level prepared statements**: Most modern drivers use the PostgreSQL wire protocol's `Parse`/`Bind`/`Execute` messages, which PgBouncer (1.21+) can handle:

```ini
# PgBouncer 1.21+
max_prepared_statements = 100  # per server connection
```

**Driver-level configuration** to avoid SQL-level PREPARE:

```python
# Python psycopg: uses protocol-level by default — no change needed
```

```javascript
// Node.js pg: protocol-level by default
const pool = new Pool({ ...config });
// For explicit control:
// statement_timeout via SET LOCAL, not SET
```

```java
// JDBC: protocol-level by default with prepareThreshold
// ?prepareThreshold=5  (default: 5 uses before server-side prepare)
// ?prepareThreshold=0  (disable server-side prepare entirely)
```

```ruby
# Ruby pg: protocol-level by default
# ActiveRecord: no special config needed
```

## Monitoring and Diagnostics

### PgBouncer Admin Console

Connect to PgBouncer's admin port:

```bash
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer
```

### Key Commands

```text
-- Pool status (most useful)
SHOW POOLS;
-- Columns: database, user, cl_active, cl_waiting, sv_active, sv_idle, sv_used, pool_mode

-- Active client and server connections
SHOW CLIENTS;
SHOW SERVERS;

-- Aggregate statistics
SHOW STATS;
-- Columns: total_xact_count, total_query_count, avg_xact_time, avg_query_time

-- Configuration
SHOW CONFIG;

-- Memory usage
SHOW MEM;
```

These commands use PgBouncer's admin protocol and fail if sent directly to PostgreSQL.

### What to Watch

| Metric | Healthy | Problem |
|--------|---------|---------|
| `cl_waiting` | 0 | > 0 = clients waiting for server connections |
| `sv_active` | < pool_size | = pool_size = pool exhausted |
| `sv_idle` | > 0 | 0 = no spare connections |
| `avg_xact_time` | < 100ms | High = long transactions hogging connections |
| `avg_wait_time` | 0 | > 0 = pool too small or transactions too long |

### Common Issues

**Clients waiting (`cl_waiting > 0`)**:
1. Check for long-running transactions: `SELECT * FROM pg_stat_activity WHERE state = 'idle in transaction';`
2. Increase `default_pool_size` (but diminishing returns beyond ~4x CPU cores)
3. Check if `query_wait_timeout` is being hit (errors in application logs)

**Server connections not being returned**:
1. `idle in transaction` sessions hold connections — set `idle_in_transaction_session_timeout` in PostgreSQL
2. Application not committing/rolling back — add explicit transaction management

## Application-Side Pooling

When PgBouncer isn't available, most drivers offer built-in connection pooling.

### Recommended Pool Sizes

**Per application instance**: 5-20 connections. Start low, increase only if you see connection wait times.

**Total across all instances**: Should not exceed ~4x database CPU cores for active connections.

### Common Driver Configuration

```python
# Python (psycopg pool)
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    conninfo="host=db port=5432 dbname=mydb",
    min_size=5,
    max_size=20,
    max_idle=300,  # close idle connections after 5 min
)

with pool.connection() as conn:
    conn.execute("SELECT ...")
```

```javascript
// Node.js (pg)
const { Pool } = require('pg');
const pool = new Pool({
    host: 'db',
    database: 'mydb',
    max: 20,           // max connections in pool
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
});

const result = await pool.query('SELECT ...');
```

```java
// Java (HikariCP)
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://db:5432/mydb");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setIdleTimeout(300000);
config.setConnectionTimeout(5000);

HikariDataSource ds = new HikariDataSource(config);
```

### Application Pool + PgBouncer

You can stack both. Each application instance pools locally (5-10 connections), and PgBouncer pools across all instances:

```
App1 (pool: 10) ──┐
App2 (pool: 10) ──┼──→ PgBouncer (pool: 30) ──→ PostgreSQL
App3 (pool: 10) ──┘
```

Set application pool sizes low when using PgBouncer — the point is to reduce total connections, not add them up.
