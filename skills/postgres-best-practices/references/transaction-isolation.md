# Transaction Isolation Reference

## Contents
- Isolation levels overview
- READ COMMITTED (default)
- REPEATABLE READ
- SERIALIZABLE
- Common pitfalls and surprises
- Choosing the right level
- Retry patterns for serialization failures

## Isolation Levels Overview

PostgreSQL implements three isolation levels (READ UNCOMMITTED maps to READ COMMITTED):

| Level | Dirty reads | Non-repeatable reads | Phantom reads | Serialization anomalies |
|-------|-------------|---------------------|---------------|------------------------|
| READ COMMITTED | No | Yes | Yes | Yes |
| REPEATABLE READ | No | No | No | Yes |
| SERIALIZABLE | No | No | No | No |

```sql
-- Check current isolation level
SHOW transaction_isolation;  -- default: 'read committed'

-- Set for a single transaction
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ... work ...
COMMIT;

-- Set for the session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set per role or database (persistent)
ALTER ROLE myapp SET default_transaction_isolation = 'repeatable read';
ALTER DATABASE mydb SET default_transaction_isolation = 'serializable';
```

## READ COMMITTED (Default)

Each statement within the transaction sees a **fresh snapshot** — it sees all data committed before that statement began, including commits by other transactions that happened after the transaction started.

### Behavior

```sql
-- Transaction A                    -- Transaction B
BEGIN;
SELECT balance FROM accounts
WHERE id = 1;  -- sees: 1000
                                    BEGIN;
                                    UPDATE accounts SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;
SELECT balance FROM accounts
WHERE id = 1;  -- sees: 500 (!)
-- The second SELECT sees B's commit
COMMIT;
```

### The Lost Update Problem

The most common surprise with READ COMMITTED — concurrent updates can silently overwrite each other:

```sql
-- Transaction A                    -- Transaction B
BEGIN;                              BEGIN;
SELECT balance FROM accounts
WHERE id = 1;  -- 1000
                                    SELECT balance FROM accounts
                                    WHERE id = 1;  -- 1000

UPDATE accounts
SET balance = 1000 - 200  -- = 800
WHERE id = 1;
                                    -- B blocks here until A commits
COMMIT;
                                    UPDATE accounts
                                    SET balance = 1000 - 300  -- = 700 (!)
                                    WHERE id = 1;
                                    COMMIT;
-- Final balance: 700
-- Expected (if serial): 500
-- A's withdrawal was lost!
```

**Fixes for lost updates in READ COMMITTED:**

1. **Use atomic SQL** — avoid read-then-write patterns:
   ```sql
   UPDATE accounts SET balance = balance - 200 WHERE id = 1;
   ```
   This is safe because the UPDATE re-evaluates `balance` from the latest committed row.

2. **Use SELECT FOR UPDATE** — lock the row before reading:
   ```sql
   BEGIN;
   SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
   -- Other transactions block here until we commit
   UPDATE accounts SET balance = balance - 200 WHERE id = 1;
   COMMIT;
   ```

3. **Use REPEATABLE READ or SERIALIZABLE** — the second transaction gets a serialization error instead of silently overwriting.

### UPDATE Re-evaluation

When an UPDATE in READ COMMITTED encounters a row locked by another transaction, it **waits** for the lock to release, then **re-evaluates** the WHERE clause against the newly committed row. If the row still matches, it proceeds with the update. If not, it skips the row.

This means UPDATE with complex WHERE clauses can behave unexpectedly:

```sql
-- Transaction A                    -- Transaction B
BEGIN;                              BEGIN;
UPDATE orders                       
SET status = 'processing'
WHERE status = 'pending'
AND id = 42;
                                    UPDATE orders
                                    SET status = 'processing'
                                    WHERE status = 'pending'
                                    AND id = 42;
                                    -- blocks, waiting for A...
COMMIT;
                                    -- A committed: row 42 is now 'processing'
                                    -- Re-evaluates: WHERE status = 'pending' → false
                                    -- UPDATE 0 (silently skips the row)
COMMIT;
```

## REPEATABLE READ

The entire transaction sees a **single snapshot** taken at the start of the first non-transaction-control statement. Subsequent statements see the same data regardless of concurrent commits.

### Behavior

```sql
-- Transaction A                    -- Transaction B
BEGIN ISOLATION LEVEL
  REPEATABLE READ;
SELECT balance FROM accounts
WHERE id = 1;  -- sees: 1000
                                    BEGIN;
                                    UPDATE accounts SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;
SELECT balance FROM accounts
WHERE id = 1;  -- still sees: 1000
COMMIT;
```

### Serialization Errors

If a REPEATABLE READ transaction tries to update a row that was modified by a concurrent committed transaction, it fails:

```sql
-- Transaction A                    -- Transaction B
BEGIN ISOLATION LEVEL
  REPEATABLE READ;
SELECT * FROM accounts
WHERE id = 1;
                                    BEGIN;
                                    UPDATE accounts SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;
UPDATE accounts
SET balance = balance - 200
WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update
```

This is **safer** than READ COMMITTED's silent re-evaluation — you know the operation failed and can retry.

### What REPEATABLE READ Does NOT Prevent

REPEATABLE READ prevents non-repeatable reads and phantoms, but does not prevent all serialization anomalies. Write skew is still possible:

```sql
-- Constraint: at least one doctor must be on-call
-- Doctor A and Doctor B are both on-call

-- Transaction A (REPEATABLE READ)     -- Transaction B (REPEATABLE READ)
BEGIN;                                  BEGIN;
SELECT count(*) FROM oncall
WHERE on_duty = true;  -- 2
                                        SELECT count(*) FROM oncall
                                        WHERE on_duty = true;  -- 2
UPDATE oncall
SET on_duty = false
WHERE doctor = 'A';   -- OK, 1 left
                                        UPDATE oncall
                                        SET on_duty = false
                                        WHERE doctor = 'B';   -- OK, 1 left
COMMIT;                                 COMMIT;
-- Both committed! No one is on-call.
```

Use SERIALIZABLE to prevent write skew.

## SERIALIZABLE

The strongest level. Transactions behave **as if they executed one at a time** (serially). PostgreSQL uses Serializable Snapshot Isolation (SSI) — an optimistic approach that detects conflicts during statement execution or at commit rather than acquiring heavy locks upfront.

### Behavior

All the anomalies prevented by REPEATABLE READ are prevented, plus serialization anomalies like write skew:

```sql
-- Same on-call scenario with SERIALIZABLE:
-- One of the two transactions will get:
-- ERROR: could not serialize access due to read/write dependencies among transactions
```

### Performance Characteristics

- Reads are not blocked — SSI tracks read/write dependencies optimistically
- Slightly higher overhead per transaction (dependency tracking)
- More serialization failures under contention (requires retry logic)
- Rarely impacts throughput in practice for OLTP workloads

### When to Use SERIALIZABLE

- Financial calculations where correctness is paramount
- Constraint enforcement that spans multiple rows or tables
- Any case where "check then act" patterns must be atomic
- When the alternative is complex application-level locking

## Common Pitfalls and Surprises

### Pitfall: Assuming READ COMMITTED Is "Safe Enough"

READ COMMITTED is safe for individual statements but not for multi-statement read-then-write patterns. Any time you SELECT a value and then use it in a subsequent UPDATE/INSERT, you have a potential race condition.

### Pitfall: Not Handling Serialization Failures

REPEATABLE READ and SERIALIZABLE can throw:
```
ERROR: could not serialize access due to concurrent update
SQLSTATE: 40001
```

**This is expected behavior, not a bug.** Applications must catch and retry.

### Pitfall: Long Transactions in REPEATABLE READ/SERIALIZABLE

The snapshot is held for the entire transaction. Long transactions:
- Prevent VACUUM from cleaning dead rows visible to the snapshot
- Increase the chance of serialization failures (more time for conflicts)
- Hold SSI dependency information longer (memory overhead)

Keep transactions short regardless of isolation level.

### Pitfall: Mixing Isolation Levels

If Transaction A is SERIALIZABLE but Transaction B is READ COMMITTED, you only get SERIALIZABLE guarantees for A's view of the data. B never sees uncommitted intermediate states, but it can observe newer committed states between statements and produce non-serializable outcomes. For full serializable behavior, **all participating transactions** must use SERIALIZABLE.

### Pitfall: DDL and Isolation

DDL statements (ALTER TABLE, CREATE INDEX) always acquire strong locks regardless of isolation level. They can block and be blocked by other transactions normally.

## Choosing the Right Level

| Scenario | Recommended level | Why |
|----------|------------------|-----|
| Simple CRUD, web apps | READ COMMITTED | Default is fine; use atomic SQL for updates |
| Balance transfers, inventory | READ COMMITTED + `SELECT FOR UPDATE` | Explicit row locking prevents lost updates |
| Reports reading consistent data | REPEATABLE READ | Snapshot consistency across multiple queries |
| Multi-row constraints, write skew | SERIALIZABLE | Only level that prevents all anomalies |
| High-contention counters | READ COMMITTED + atomic UPDATE | `UPDATE x SET n = n + 1` is inherently safe |

**Default to READ COMMITTED** and escalate only when you identify a specific anomaly your application cannot tolerate.

## Retry Patterns for Serialization Failures

### Application-Level Retry (Recommended)

```python
# Python example (psycopg)
import random
import time

import psycopg
from psycopg.errors import DeadlockDetected, SerializationFailure

MAX_RETRIES = 5
BASE_DELAY_SECONDS = 0.01
MAX_DELAY_SECONDS = 0.5

for attempt in range(MAX_RETRIES):
    try:
        # This must be a top-level transaction, not a savepoint inside one.
        with conn.transaction():
            conn.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
            conn.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
            conn.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
        break  # success
    except (SerializationFailure, DeadlockDetected):
        if attempt == MAX_RETRIES - 1:
            raise  # give up after max retries

        # Bounded exponential backoff with jitter reduces repeated collisions.
        delay = min(MAX_DELAY_SECONDS, BASE_DELAY_SECONDS * (2 ** attempt))
        time.sleep(random.uniform(delay / 2, delay))
```

### Key Retry Rules

1. **Retry the entire transaction** — not just the failed statement
2. **Re-run all reads and decision-making** — transaction inputs may no longer be valid after a conflict
3. **Use a bounded retry count** — avoid infinite retry loops
4. **Use bounded exponential backoff with jitter** — concurrent retriers should not repeatedly collide; tune the delays for the workload
5. **Log retries and exhaustion** — frequent retries indicate contention or transaction-design problems
6. **SQLSTATE 40001** is the error code to catch (`serialization_failure`)
7. **SQLSTATE 40P01** is deadlock (`deadlock_detected`) — it can be retried, but also fix inconsistent lock ordering where possible
8. **Do not publish external side effects before commit** — messages, emails, and API calls can otherwise be duplicated by a retry

PostgreSQL requires retrying the complete transaction and warns that multiple attempts may be needed. See [Serialization Failure Handling](https://www.postgresql.org/docs/current/mvcc-serialization-failure-handling.html).

### PL/pgSQL Procedure for the Transaction Body

A PL/pgSQL function cannot retry a complete transaction. An `EXCEPTION` block rolls back only its subtransaction; retrying the block remains inside the same top-level transaction and uses the same transaction snapshot at REPEATABLE READ or SERIALIZABLE.

A procedure can encapsulate the database work, but the application should still own the complete transaction and retry loop:

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_id int,
    p_to_id int,
    p_amount numeric
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts
    SET balance = balance - p_amount
    WHERE id = p_from_id;

    UPDATE accounts
    SET balance = balance + p_amount
    WHERE id = p_to_id;
END;
$$;
```

Invoke `CALL transfer_funds(...)` inside the application-managed transaction and retry that entire transaction from the application. A serialization failure can occur at commit, so a procedure cannot reliably catch every failure and restart itself.
