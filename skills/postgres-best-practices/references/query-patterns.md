# SQL Query Patterns Reference

## Contents
- Common Table Expressions (CTEs)
- Window functions
- Lateral joins
- Recursive queries
- UPSERT (INSERT ON CONFLICT)
- Bulk operations
- JSONB queries
- Date/time patterns
- Anti-patterns to avoid

## Common Table Expressions (CTEs)

### Readability CTEs

```sql
WITH active_customers AS (
    SELECT id, name, email
    FROM customers
    WHERE status = 'active'
),
recent_orders AS (
    SELECT customer_id, count(*) AS order_count, max(created_at) AS last_order
    FROM orders
    WHERE created_at > now() - interval '90 days'
    GROUP BY customer_id
)
SELECT ac.name, ac.email, ro.order_count, ro.last_order
FROM active_customers ac
JOIN recent_orders ro ON ro.customer_id = ac.id
ORDER BY ro.order_count DESC;
```

### CTE Materialization

By default the optimizer may inline CTEs. Force materialization when:
- The CTE is referenced multiple times
- You want to create an optimization fence

```sql
WITH expensive_calc AS MATERIALIZED (
    SELECT ... -- complex aggregation
)
SELECT * FROM expensive_calc WHERE ...
UNION ALL
SELECT * FROM expensive_calc WHERE ...;
```

Force inlining (default for single-use):
```sql
WITH simple_filter AS NOT MATERIALIZED (
    SELECT * FROM large_table WHERE status = 'active'
)
SELECT * FROM simple_filter WHERE created_at > '2024-01-01';
```

## Window Functions

### Ranking

```sql
-- Row number (no ties)
SELECT *, row_number() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
FROM employees;

-- Rank (ties get same rank, gaps after)
SELECT *, rank() OVER (ORDER BY score DESC) AS rank
FROM leaderboard;

-- Dense rank (no gaps)
SELECT *, dense_rank() OVER (ORDER BY score DESC) AS dense_rank
FROM leaderboard;
```

### Running Totals and Moving Averages

```sql
SELECT
    date,
    revenue,
    sum(revenue) OVER (ORDER BY date) AS running_total,
    avg(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_revenue;
```

### Lead/Lag (Access Adjacent Rows)

```sql
SELECT
    event_time,
    event_type,
    lag(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event,
    event_time - lag(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS time_since_last
FROM user_events;
```

### First/Last Value

```sql
SELECT DISTINCT
    department,
    first_value(name) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS highest_paid
FROM employees;
```

### Gaps and Islands

Find consecutive sequences:

```sql
WITH numbered AS (
    SELECT *,
        date - (row_number() OVER (PARTITION BY user_id ORDER BY date))::int * interval '1 day' AS grp
    FROM daily_logins
)
SELECT user_id, min(date) AS streak_start, max(date) AS streak_end,
       count(*) AS streak_length
FROM numbered
GROUP BY user_id, grp
HAVING count(*) >= 7;  -- streaks of 7+ days
```

## Lateral Joins

LATERAL lets a subquery reference columns from preceding tables. Essential for "top-N per group" queries.

### Top-N Per Group

```sql
-- Get latest 3 orders per customer
SELECT c.name, o.*
FROM customers c
CROSS JOIN LATERAL (
    SELECT id, total, created_at
    FROM orders
    WHERE customer_id = c.id
    ORDER BY created_at DESC
    LIMIT 3
) o;
```

This is much faster than window function approaches when you have an index on `orders(customer_id, created_at DESC)`.

### Calling Set-Returning Functions

```sql
SELECT j.id, elem.key, elem.value
FROM journal_entries j
CROSS JOIN LATERAL jsonb_each_text(j.metadata) AS elem;
```

## Recursive Queries

### Tree Traversal (Adjacency List)

```sql
WITH RECURSIVE tree AS (
    -- Base case: root nodes
    SELECT id, name, parent_id, 0 AS depth, ARRAY[id] AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive step
    SELECT c.id, c.name, c.parent_id, t.depth + 1, t.path || c.id
    FROM categories c
    JOIN tree t ON t.id = c.parent_id
    WHERE NOT c.id = ANY(t.path)  -- cycle prevention
)
SELECT * FROM tree ORDER BY path;
```

### Generating Series

```sql
-- Date series for gap-filling
SELECT d::date, coalesce(o.total, 0) AS total
FROM generate_series('2024-01-01'::date, '2024-12-31'::date, '1 day') AS d
LEFT JOIN (
    SELECT created_at::date AS day, sum(total) AS total
    FROM orders
    GROUP BY 1
) o ON o.day = d;
```

## UPSERT (INSERT ON CONFLICT)

### Basic Upsert

```sql
INSERT INTO user_settings (user_id, key, value)
VALUES (1, 'theme', 'dark')
ON CONFLICT (user_id, key)
DO UPDATE SET
    value = EXCLUDED.value,
    updated_at = now();
```

### Insert-or-Ignore

```sql
INSERT INTO tags (name)
VALUES ('postgres'), ('sql'), ('database')
ON CONFLICT (name) DO NOTHING;
```

### Upsert with Conditional Update

```sql
INSERT INTO inventory (sku, quantity, last_restock)
VALUES ('ABC-123', 50, now())
ON CONFLICT (sku)
DO UPDATE SET
    quantity = inventory.quantity + EXCLUDED.quantity,
    last_restock = EXCLUDED.last_restock
WHERE inventory.quantity < 100;  -- only restock if low
```

### Upsert Returning

```sql
INSERT INTO users (email, name)
VALUES ('a@b.com', 'Alice')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name
RETURNING id;
```

Do not use `(xmax = 0)` as an inserted-versus-updated contract; it relies on implementation details. On PG18+, use `OLD`/`NEW` in `RETURNING`. On earlier versions, distinguish outcomes in application logic or use separate statements when the distinction matters.

### OLD/NEW in RETURNING (PG18+)

All DML commands support `OLD` and `NEW` references in RETURNING, making it easy to see before/after values:

```sql
UPDATE products SET price = price * 1.10
WHERE category = 'electronics'
RETURNING id, old.price AS old_price, new.price AS new_price;

DELETE FROM sessions WHERE expired_at < now()
RETURNING old.*;
```

## MERGE (PG15+)

SQL-standard command for conditional INSERT, UPDATE, or DELETE in a single statement. Replaces UPSERT for complex cases where you need different actions based on whether the row exists.

```sql
MERGE INTO inventory AS target
USING incoming_shipment AS source
ON target.sku = source.sku
WHEN MATCHED AND source.quantity = 0 THEN
    DELETE
WHEN MATCHED THEN
    UPDATE SET quantity = target.quantity + source.quantity,
               last_restock = now()
WHEN NOT MATCHED THEN
    INSERT (sku, quantity, last_restock)
    VALUES (source.sku, source.quantity, now());
```

### MERGE RETURNING (PG17+)

```sql
MERGE INTO inventory AS target
USING incoming_shipment AS source
ON target.sku = source.sku
WHEN MATCHED THEN
    UPDATE SET quantity = target.quantity + source.quantity
WHEN NOT MATCHED THEN
    INSERT (sku, quantity) VALUES (source.sku, source.quantity)
RETURNING merge_action(), target.*;
-- merge_action() returns 'INSERT', 'UPDATE', or 'DELETE'
```

**When to use MERGE vs UPSERT**: Use `INSERT ON CONFLICT` for simple insert-or-update on a single table. Use `MERGE` when you need different actions based on conditions, when the source is another table/query, or when you need DELETE as one of the outcomes.

## JSON_TABLE (PG17+)

Convert JSON data to a relational table, usable in FROM:

```sql
SELECT jt.*
FROM events,
     JSON_TABLE(
         payload, '$'
         COLUMNS (
             event_type text PATH '$.type',
             city text PATH '$.address.city',
             score int PATH '$.score'
         )
     ) AS jt
WHERE jt.event_type = 'click';
```

## Bulk Operations

### Bulk Insert from Values

```sql
INSERT INTO products (name, price, category)
VALUES
    ('Widget A', 9.99, 'gadgets'),
    ('Widget B', 14.99, 'gadgets'),
    ('Widget C', 19.99, 'gadgets');
```

### Bulk Update with FROM

```sql
UPDATE products p
SET price = new_prices.price
FROM (VALUES
    (1, 12.99),
    (2, 15.99),
    (3, 22.99)
) AS new_prices(id, price)
WHERE p.id = new_prices.id;
```

### Bulk Delete with Subquery

```sql
DELETE FROM sessions
WHERE id IN (
    SELECT id FROM sessions
    WHERE last_active < now() - interval '30 days'
    ORDER BY last_active
    LIMIT 10000  -- batch to avoid long locks
);
```

## JSONB Queries

### Querying JSONB

```sql
-- Key access
SELECT payload->>'name' AS name FROM events;             -- text
SELECT payload->'address'->'city' FROM events;            -- jsonb
SELECT payload#>>'{address,city}' FROM events;            -- text via path

-- Containment (uses GIN index)
SELECT * FROM events WHERE payload @> '{"type": "click"}';

-- Key existence
SELECT * FROM events WHERE payload ? 'error_code';

-- Array element access
SELECT payload->'tags'->>0 FROM events;
```

### JSONB Aggregation

```sql
-- Build JSON object from rows
SELECT jsonb_object_agg(key, value) FROM settings WHERE user_id = 1;

-- Build JSON array from rows
SELECT jsonb_agg(jsonb_build_object('id', id, 'name', name))
FROM products
WHERE category = 'gadgets';
```

### JSONB Modification

```sql
-- Set/overwrite a key
UPDATE events SET payload = payload || '{"processed": true}';

-- Remove a key
UPDATE events SET payload = payload - 'temp_field';

-- Set nested key
UPDATE events SET payload = jsonb_set(payload, '{status,code}', '"200"');
```

## Date/Time Patterns

### JSONB Subscripting (PG14+)

Simpler syntax for JSONB access and assignment:

```sql
-- Read JSONB (equivalent to payload->'address'->'city')
SELECT payload['address']['city'] FROM events;

-- Update (equivalent to jsonb_set)
UPDATE events SET payload['processed'] = 'true' WHERE id = 1;
UPDATE events SET payload['address']['zip'] = '"90210"' WHERE id = 1;
```

### date_bin() (PG14+)

Bin timestamps into uniform intervals — more flexible than `date_trunc`:

```sql
-- 15-minute bins (date_trunc can only do hour/day/etc.)
SELECT
    date_bin('15 minutes', created_at, '2024-01-01') AS bin,
    count(*)
FROM orders
GROUP BY 1 ORDER BY 1;

-- 6-hour bins
SELECT date_bin('6 hours', occurred_at, '2024-01-01') AS bin, count(*)
FROM events GROUP BY 1;
```

### Truncation and Grouping

```sql
SELECT
    date_trunc('month', created_at) AS month,
    count(*) AS orders
FROM orders
GROUP BY 1
ORDER BY 1;
```

### Interval Arithmetic

```sql
SELECT * FROM subscriptions
WHERE expires_at BETWEEN now() AND now() + interval '7 days';
```

### Extract Components

```sql
SELECT
    extract(dow FROM created_at) AS day_of_week,  -- 0=Sun, 6=Sat
    extract(hour FROM created_at) AS hour,
    count(*)
FROM orders
GROUP BY 1, 2;
```

## Concurrency Patterns

### SKIP LOCKED (Queue Processing)

Multiple workers can process a queue table without blocking each other:

```sql
-- Worker claims and processes next N items atomically
WITH next_batch AS (
    SELECT id FROM job_queue
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue SET status = 'processing', started_at = now()
WHERE id IN (SELECT id FROM next_batch)
RETURNING *;
```

### Deadlock Prevention

Always acquire row locks in a consistent order (e.g., by ID):

```sql
-- Bad: two transactions locking rows in different order → deadlock
-- Good: always lock in ID order
SELECT * FROM accounts
WHERE id IN (5, 12, 3)
ORDER BY id
FOR UPDATE;
```

### Transaction Discipline

```sql
-- Set statement timeout to prevent runaway queries
SET statement_timeout = '30s';

-- Kill idle-in-transaction sessions (holds locks, blocks VACUUM)
SET idle_in_transaction_session_timeout = '60s';

-- Limit total transaction duration (PG17+)
SET transaction_timeout = '5min';
```

Keep transactions short: move external calls (HTTP, file I/O) outside the transaction boundary.

### N+1 Query Elimination

```sql
-- Bad: one query per item (N+1 pattern)
-- SELECT * FROM orders WHERE customer_id = 1;
-- SELECT * FROM orders WHERE customer_id = 2;
-- ...

-- Good: single query with ANY
SELECT * FROM orders WHERE customer_id = ANY($1::bigint[]);
-- Pass array of IDs: ARRAY[1, 2, 3, ...]
```

## Useful PG18+ Functions

### array_sort() and array_reverse()

```sql
SELECT array_sort(ARRAY[3, 1, 4, 1, 5]);  -- {1, 1, 3, 4, 5}
SELECT array_reverse(ARRAY[1, 2, 3]);      -- {3, 2, 1}
```

### casefold() — Unicode Case-Insensitive Matching

More robust than `lower()` for internationalized text:

```sql
SELECT casefold('Hello World') = casefold('HELLO WORLD');  -- true
```

## Anti-Patterns to Avoid

### SELECT * in Production Queries
Use explicit column lists. `SELECT *` breaks when columns change and prevents index-only scans.

### NOT IN with NULLs
`NOT IN (subquery)` returns no rows if any subquery result is NULL. Use `NOT EXISTS` instead:

```sql
-- Bad: breaks if orders.customer_id has any NULL
SELECT * FROM customers WHERE id NOT IN (SELECT customer_id FROM orders);

-- Good: always correct
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

### Implicit Casts in WHERE Clauses
Casting a column prevents index use:

```sql
-- Bad: casts every row, no index
WHERE created_at::date = '2024-01-15'

-- Good: range scan, uses index
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16'
```

### ORDER BY on Unindexed Large Result Sets
If you ORDER BY + LIMIT on a large table, ensure there's an index on the ORDER BY columns to avoid a full sort.

### Using OFFSET for Pagination
OFFSET scans and discards rows. Use keyset pagination instead:

```sql
-- Bad: slow at high offsets
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 10000;

-- Good: keyset pagination
SELECT * FROM products WHERE id > $last_seen_id ORDER BY id LIMIT 20;
```
