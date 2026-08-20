# Schema Design Reference

## Contents
- Data type best practices
- Primary keys
- Foreign keys and referential integrity
- Check and exclusion constraints
- Normalization guidelines
- Denormalization patterns
- Partitioning
- Multi-tenant patterns
- Migration safety

## Data Type Best Practices

### Identity Columns

```sql
-- Preferred: GENERATED ALWAYS
CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ...
);

-- If you need to occasionally override:
INSERT INTO orders OVERRIDING SYSTEM VALUE ...

-- UUID alternative (good for distributed systems)
CREATE TABLE orders (
    id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
    ...
);
```

**UUID v4 vs v7**: `gen_random_uuid()` generates UUIDv4 (random), which causes B-tree index fragmentation on large tables because inserts scatter across the index. UUIDv7 (time-ordered) preserves insert locality. **PG18+** has built-in `uuidv7()` (and explicit `uuidv4()`). On older versions, use the `pg_uuidv7` extension. Prefer UUIDv7 for primary keys.

```sql
-- PG18+: built-in UUIDv7
CREATE TABLE events (
    id uuid DEFAULT uuidv7() PRIMARY KEY,
    ...
);
```

Avoid `serial` / `bigserial` — they create an implicit sequence with looser ownership semantics.

### Timestamps

Always use `timestamptz`. PostgreSQL stores it as UTC internally and converts on display.

```sql
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    occurred_at timestamptz NOT NULL DEFAULT now(),
    -- NOT: occurred_at timestamp  (loses timezone context)
);
```

### Text Fields

Prefer `text` with a `CHECK` constraint over `varchar(n)`:

```sql
CREATE TABLE users (
    email text NOT NULL CHECK (length(email) <= 254),
    username text NOT NULL CHECK (length(username) BETWEEN 3 AND 30)
);
```

`varchar(n)` provides no performance benefit and makes future changes harder.

### UNIQUE NULLS NOT DISTINCT (PG15+)

By default, NULLs are considered distinct in unique constraints (multiple NULLs allowed). PG15 adds an option to treat NULLs as equal:

```sql
-- Standard: allows multiple rows with NULL in email
CREATE UNIQUE INDEX idx_email_standard ON users(email);

-- PG15+: only one NULL allowed
CREATE UNIQUE INDEX idx_email_nulls_not_distinct
    ON users(email) NULLS NOT DISTINCT;
```

### Naming Conventions

Use `snake_case` without quotes for all identifiers. Unquoted identifiers fold to lowercase automatically. Quoted mixed-case identifiers (e.g., `"userId"`) require quotes everywhere and break many ORMs, tools, and AI assistants.

### Enums vs. Check Constraints vs. Lookup Tables

| Approach | Pros | Cons |
|----------|------|------|
| `CHECK (col IN (...))` | Simple, no DDL type | Requires ALTER TABLE to add values |
| `CREATE TYPE ... AS ENUM` | Type safety, compact storage | Cannot remove values, ALTER TYPE needed |
| Lookup/reference table | FK enforced, can add metadata | Extra join |

For small, stable sets (status, priority): CHECK or ENUM.
For evolving sets or sets needing metadata: lookup table.

## Foreign Keys

### Basics

```sql
CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL REFERENCES customers(id),
    ...
);
```

### ON DELETE / ON UPDATE Actions

| Action | Use when |
|--------|----------|
| `NO ACTION` (default) | Reject the update or deletion if a violation remains when the constraint is checked; the check can be deferred when the foreign key is deferrable and currently deferred |
| `RESTRICT` | Reject the referenced operation immediately; the check cannot be deferred |
| `CASCADE` | Child rows are meaningless without parent (e.g., order_items when order is deleted) |
| `SET NULL` | Relationship is optional, preserve child row |
| `SET DEFAULT` | Rare; reassign to a valid default parent |

### Deferrable Foreign Keys

Useful when you need to insert rows in both tables within a single transaction regardless of order:

```sql
ALTER TABLE order_items
    ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id)
    DEFERRABLE INITIALLY DEFERRED;
```

### Index the FK Column

PostgreSQL does NOT auto-create indexes on FK columns. Always add one:

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

Without this index:
- JOINs on the FK are slow (seq scan on child table)
- DELETE on the parent table locks the child table and scans it fully

## Check Constraints

```sql
CREATE TABLE products (
    price numeric(10,2) NOT NULL CHECK (price > 0),
    discount_pct numeric(3,2) CHECK (discount_pct BETWEEN 0 AND 1),
    start_date date NOT NULL,
    end_date date,
    CHECK (end_date IS NULL OR end_date > start_date)
);
```

### Exclusion Constraints (Prevent Overlaps)

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    room_id int NOT NULL,
    during tstzrange NOT NULL,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

This prevents overlapping bookings for the same room at the database level.

### Temporal Constraints — WITHOUT OVERLAPS (PG18+)

PG18 adds built-in temporal constraint support, simplifying the exclusion constraint pattern above:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    room_id int NOT NULL,
    during tstzrange NOT NULL,
    PRIMARY KEY (room_id, during WITHOUT OVERLAPS)
);
```

Foreign keys can also reference temporal primary keys using `PERIOD`:

```sql
CREATE TABLE reservations (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id int NOT NULL,
    during tstzrange NOT NULL,
    FOREIGN KEY (room_id, PERIOD during)
        REFERENCES room_bookings (room_id, PERIOD during)
);
```

This eliminates the need to write the exclusion constraint manually. The temporal key is implemented with GiST; scalar key columns such as `room_id int` still need an appropriate GiST operator class, supplied by `btree_gist` for common scalar types.

### NOT ENFORCED Constraints (PG18+)

CHECK and foreign key constraints can be marked as informational only — the database trusts the data without enforcing the constraint. Useful for documenting intent or helping the planner without paying enforcement cost:

```sql
ALTER TABLE orders ADD CONSTRAINT positive_total
    CHECK (total > 0) NOT ENFORCED;
```

## Normalization Guidelines

### When to Normalize

- Data integrity is critical (financial, compliance)
- Multiple writers update the same logical data
- Storage is a concern (avoid data duplication)

### When to Denormalize

- Read-heavy workloads where join cost is measurable
- Materialized views can serve as denormalized read models
- JSONB columns for flexible, schema-less attributes that don't need FK integrity

### Materialized Views as Denormalization

```sql
CREATE MATERIALIZED VIEW mv_order_summary AS
SELECT
    o.id,
    o.created_at,
    c.name AS customer_name,
    sum(oi.quantity * oi.unit_price) AS total
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.created_at, c.name;

CREATE UNIQUE INDEX ON mv_order_summary(id);

-- Refresh periodically or after batch writes:
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_summary;
```

## Partitioning (Declarative)

### When to Partition

- Table exceeds tens of millions of rows
- Queries consistently filter on the partition key
- You need efficient bulk deletion (DROP partition vs. DELETE)
- Maintenance operations (VACUUM, reindex) are slow on the full table

### Range Partitioning (most common)

```sql
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY,
    occurred_at timestamptz NOT NULL,
    payload jsonb
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

### List Partitioning (multi-tenant)

```sql
CREATE TABLE tenant_data (
    tenant_id int NOT NULL,
    data jsonb
) PARTITION BY LIST (tenant_id);

CREATE TABLE tenant_data_1 PARTITION OF tenant_data FOR VALUES IN (1);
CREATE TABLE tenant_data_2 PARTITION OF tenant_data FOR VALUES IN (2);
```

### Detaching Partitions (PG14+)

```sql
-- Non-blocking detach (PG14+): doesn't hold AccessExclusiveLock
ALTER TABLE events DETACH PARTITION events_2024_q1 CONCURRENTLY;
```

### Key Rules

- Partition key must be part of the primary key and all unique indexes
- Indexes defined on parent are auto-created on child partitions
- Foreign keys referencing partitioned tables are supported
- Exclusion constraints on partitioned tables require PG17+ (equality on partition key only)

## Virtual Generated Columns (PG18+)

PG18 defaults generated columns to `VIRTUAL` (computed at read time, not stored on disk). Use `STORED` when you need to index the expression.

```sql
CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    quantity int NOT NULL,
    unit_price numeric(10,2) NOT NULL,
    description text NOT NULL,
    -- Virtual: computed on read, no storage cost
    total numeric GENERATED ALWAYS AS (quantity * unit_price) VIRTUAL,
    -- Stored: persisted on disk, can be indexed
    search_text tsvector GENERATED ALWAYS AS (
        to_tsvector('english', description)
    ) STORED
);
```

Virtual columns save storage and write I/O but cannot be indexed directly. Use `STORED` when you need an index on the expression.

## Multi-Tenant Patterns

### Row-Level Security (RLS)

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::int);

-- Set per-connection:
SET app.current_tenant = '42';
```

### Shared Schema with tenant_id

- Add `tenant_id` to every table
- Include `tenant_id` in all indexes (composite with other columns)
- Use RLS or application-level filtering

## Migration Safety

### Safe Column Operations

| Operation | Safe online? | Notes |
|-----------|-------------|-------|
| `ADD COLUMN` (nullable, no default) | Yes | Instant metadata change |
| `ADD COLUMN ... DEFAULT x` | Yes | Default stored in catalog, no table rewrite |
| `DROP COLUMN` | Yes | Marks column as dropped, no rewrite |
| `ALTER COLUMN SET NOT NULL` | Caution | Full table scan to validate (use CHECK first) |
| `ALTER COLUMN TYPE` | No | Full table rewrite + exclusive lock |
| `ADD CONSTRAINT ... NOT VALID` | Yes | Doesn't scan existing rows |
| `VALIDATE CONSTRAINT` | Yes | ShareUpdateExclusiveLock (reads/writes allowed) |

### Safe NOT NULL Pattern

```sql
-- Step 1: Add CHECK constraint without validating existing rows
ALTER TABLE orders ADD CONSTRAINT orders_status_nn
    CHECK (status IS NOT NULL) NOT VALID;

-- Step 2: Validate in background (non-blocking)
ALTER TABLE orders VALIDATE CONSTRAINT orders_status_nn;

-- Step 3: SET NOT NULL is instant if a valid CHECK exists
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;

-- Step 4: Drop the now-redundant CHECK
ALTER TABLE orders DROP CONSTRAINT orders_status_nn;
```
