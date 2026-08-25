# Security & Roles Reference

## Contents
- Role management
- Privilege system
- Schema-based access control
- Row-Level Security (RLS)
- pg_hba.conf authentication
- Password policies and authentication methods
- Security functions and best practices

## Role Management

PostgreSQL uses **roles** for both users and groups. A role with `LOGIN` is a user; a role without is a group.

### Creating Roles

```sql
-- User role (can log in)
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password_here';

-- Group role (cannot log in, used for privilege grouping)
CREATE ROLE readonly;
CREATE ROLE readwrite;
CREATE ROLE admin;

-- Role with specific attributes
CREATE ROLE backup_user WITH LOGIN REPLICATION PASSWORD '...';
CREATE ROLE migrator WITH LOGIN CREATEDB PASSWORD '...';
```

### Role Attributes

| Attribute | Meaning |
|-----------|---------|
| `LOGIN` | Can connect to the database |
| `SUPERUSER` | Bypasses all permission checks (dangerous) |
| `CREATEDB` | Can create databases |
| `CREATEROLE` | Can create/alter/drop other roles |
| `REPLICATION` | Can initiate streaming replication |
| `BYPASSRLS` | Bypasses Row-Level Security policies |
| `INHERIT` | Automatically inherits privileges of member roles (default) |
| `CONNECTION LIMIT n` | Max concurrent connections for this role |
| `VALID UNTIL 'timestamp'` | Password expiration |

```sql
-- Modify attributes
ALTER ROLE app_user WITH CONNECTION LIMIT 10;
ALTER ROLE temp_user VALID UNTIL '2025-01-01';
ALTER ROLE app_user WITH PASSWORD 'new_password';

-- Inspect roles
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin,
       rolreplication, rolbypassrls, rolconnlimit, rolvaliduntil
FROM pg_roles
WHERE rolname NOT LIKE 'pg_%'
ORDER BY rolname;
```

### Group Membership

```sql
-- Add a user to a group
GRANT readonly TO app_user;
GRANT readwrite TO app_user;

-- Remove membership
REVOKE readwrite FROM app_user;

-- Check memberships
SELECT
    r.rolname AS role,
    m.rolname AS member
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
ORDER BY r.rolname, m.rolname;
```

### Predefined Roles

PostgreSQL provides system-defined roles for common privileged capabilities. PostgreSQL 13 and earlier documentation called these "default roles"; PostgreSQL 14 renamed the category to "predefined roles." Individual roles were introduced in different releases.

| Role | Grants |
|------|--------|
| `pg_read_all_data` | SELECT on all tables, views, sequences in all schemas |
| `pg_write_all_data` | INSERT, UPDATE, DELETE on all tables, sequences in all schemas |
| `pg_read_all_settings` | Read all GUC settings (even superuser-only) |
| `pg_read_all_stats` | Read all pg_stat_* views |
| `pg_monitor` | Read monitoring views (`pg_stat_*`, `pg_locks`, etc.) |
| `pg_signal_backend` | Send signals to other backends (cancel/terminate) |
| `pg_checkpoint` (PG15+) | Run CHECKPOINT |
| `pg_maintain` (PG17+) | Run VACUUM, ANALYZE, REINDEX, CLUSTER, REFRESH MATERIALIZED VIEW, and LOCK TABLE on all relations |

```sql
-- Give a monitoring role read access to all stats
GRANT pg_monitor TO monitoring_user;

-- Give an app role read access to all data without per-table grants
GRANT pg_read_all_data TO reporting_user;
```

## Privilege System

### Object Privileges

```sql
-- Grant on tables
GRANT SELECT ON orders TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO readwrite;
GRANT ALL PRIVILEGES ON orders TO admin;

-- Grant on all tables in a schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;

-- Grant on sequences (needed for INSERT with serial/identity columns)
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO readwrite;

-- Grant on schema (required to access objects within it)
GRANT USAGE ON SCHEMA public TO readonly;
GRANT USAGE, CREATE ON SCHEMA public TO readwrite;
```

### Default Privileges

Set privileges that automatically apply to future objects:

```sql
-- As the object owner or superuser:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO readwrite;

-- For objects created by a specific role:
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;
```

**Important**: Default privileges only affect objects created **after** the `ALTER DEFAULT PRIVILEGES` command. Existing objects need explicit `GRANT`.

### Revoking Privileges

```sql
-- Revoke specific privileges
REVOKE INSERT, UPDATE, DELETE ON orders FROM readonly;

-- Revoke all privileges
REVOKE ALL PRIVILEGES ON orders FROM some_role;

-- Revoke from public (default grant on new databases)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
```

### Inspecting Privileges

```sql
-- Table privileges
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.table_privileges
WHERE table_schema = 'public'
ORDER BY table_name, grantee;

-- Compact: privileges per table
SELECT
    relname,
    relacl  -- array of 'grantee=privileges/grantor'
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace;

-- Function privileges
SELECT
    routine_name,
    grantee,
    privilege_type
FROM information_schema.routine_privileges
WHERE routine_schema = 'public';
```

## Schema-Based Access Control

Use schemas to organize objects and control access:

```sql
-- Create isolated schemas
CREATE SCHEMA app;
CREATE SCHEMA reporting;
CREATE SCHEMA staging;

-- Grant access per schema
GRANT USAGE ON SCHEMA app TO app_user;
GRANT USAGE ON SCHEMA reporting TO reporting_user;
GRANT USAGE ON SCHEMA staging TO etl_user;

-- Remove default public schema access
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

### Search Path Security

The `search_path` determines which schemas are searched for unqualified table names. A malicious `search_path` can redirect queries to attacker-controlled tables.

```sql
-- Set a safe search path (explicit, no reliance on public)
ALTER ROLE app_user SET search_path = app, public;

-- For functions: use SECURITY DEFINER carefully
CREATE FUNCTION get_balance(account_id int) RETURNS numeric
    SECURITY DEFINER
    SET search_path = pg_catalog, pg_temp
AS $$
    SELECT balance FROM app.accounts WHERE id = account_id;
$$ LANGUAGE sql;
```

**Always give `SECURITY DEFINER` functions a fixed `search_path` containing only trusted schemas.** Explicitly list `pg_temp` last: if it is omitted, PostgreSQL searches the session's temporary schema first for relations and data types, which can allow temporary objects to shadow intended objects. When application objects are fully qualified, prefer `pg_catalog, pg_temp`; otherwise list only trusted application schemas followed by `pg_temp`.

## Row-Level Security (RLS)

RLS adds per-row access control enforced by the database, not the application.

### Basic Setup

```sql
-- Enable RLS on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy for tenant isolation
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::int);

-- Test through a non-owner application role; table owners bypass RLS by default.
SET ROLE app_user;
SET app.current_tenant = '42';

-- All queries on orders now automatically filter by tenant_id = 42
SELECT * FROM orders;  -- only sees tenant 42's orders
RESET ROLE;
```

### Policy Types

```sql
-- SELECT policy (restrict which rows can be read)
CREATE POLICY read_own ON documents
    FOR SELECT
    USING (owner_id = current_setting('app.current_user_id')::bigint);

-- INSERT policy (restrict which rows can be inserted)
CREATE POLICY insert_own ON documents
    FOR INSERT
    WITH CHECK (owner_id = current_setting('app.current_user_id')::bigint);

-- UPDATE policy (restrict which rows can be updated, and what values are allowed)
CREATE POLICY update_own ON documents
    FOR UPDATE
    USING (owner_id = current_setting('app.current_user_id')::bigint)       -- rows selectable for update
    WITH CHECK (owner_id = current_setting('app.current_user_id')::bigint); -- required post-update value

-- DELETE policy
CREATE POLICY delete_own ON documents
    FOR DELETE
    USING (owner_id = current_setting('app.current_user_id')::bigint);

-- ALL (applies to all commands)
CREATE POLICY full_access ON documents
    FOR ALL
    USING (owner_id = current_setting('app.current_user_id')::bigint)
    WITH CHECK (owner_id = current_setting('app.current_user_id')::bigint);
```

### Multiple Policies

Multiple policies on the same table are combined with OR (by default). Use `RESTRICTIVE` for AND:

```sql
-- Permissive (default): combined with OR
CREATE POLICY see_active ON orders
    USING (status = 'active');

CREATE POLICY see_own ON orders
    USING (user_id = current_setting('app.current_user_id')::bigint);
-- User sees rows that are active OR owned by them

-- Restrictive: combined with AND (with permissive policies)
CREATE POLICY must_be_active ON orders AS RESTRICTIVE
    USING (status != 'deleted');
-- Combined: (active OR own) AND not_deleted
```

### RLS Caveats

- **Table owner bypasses RLS** by default. Use `ALTER TABLE ... FORCE ROW LEVEL SECURITY` to apply RLS to the owner too.
- **Superusers and `BYPASSRLS` roles** bypass all RLS policies.
- **No policies = deny all** when RLS is enabled (except for table owner).
- **Performance**: RLS adds filter conditions to every query. Ensure the policy columns are indexed.
- **Leaky views**: Functions in RLS policies that are not `LEAKPROOF` could theoretically leak data via error messages or side channels. Use `LEAKPROOF` functions where possible.

```sql
-- Force RLS on table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Check which tables have RLS enabled
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace;
```

## pg_hba.conf Authentication

`pg_hba.conf` controls who can connect, from where, and how they authenticate. Rules are evaluated top-to-bottom; first match wins.

### Format

```
# TYPE    DATABASE    USER        ADDRESS         METHOD
local     all         all                         scram-sha-256
host      all         all         127.0.0.1/32    scram-sha-256
host      all         all         ::1/128         scram-sha-256
host      mydb        app_user    10.0.0.0/8      scram-sha-256
host      all         all         0.0.0.0/0       reject
```

### Authentication Methods

| Method | Security | Use case |
|--------|----------|----------|
| `scram-sha-256` | Strong | **Recommended default** — salted challenge-response |
| `md5` | Weak | Legacy — **deprecated in PG18** (emits warnings) |
| `cert` | Strong | Client certificate (mutual TLS) |
| `peer` | Strong | Local connections — maps OS user to PG role |
| `ident` | Moderate | TCP — maps OS user via ident server |
| `gss` | Strong | Kerberos/GSSAPI |
| `ldap` | Moderate | LDAP directory authentication |
| `trust` | None | **Never use in production** — no password required |
| `reject` | N/A | Explicitly deny connections |

### Best Practices

```
# 1. Use scram-sha-256 (not md5)
# 2. Be specific — don't use 'all' for database/user in production
# 3. Restrict by IP range
# 4. Put reject rules at the bottom as a catch-all
# 5. Use 'local peer' for admin access (no password over unix socket)

local     all         postgres                    peer
host      mydb        app_user    10.0.1.0/24     scram-sha-256
host      replication repl_user   10.0.2.0/24     scram-sha-256
hostssl   mydb        all         0.0.0.0/0       scram-sha-256
host      all         all         0.0.0.0/0       reject
```

After modifying `pg_hba.conf`, reload:

```sql
SELECT pg_reload_conf();
-- or from CLI: pg_ctl reload
```

## Password Policies and Authentication

### Enforce scram-sha-256

```sql
-- In postgresql.conf:
-- password_encryption = scram-sha-256    (default in PG14+)

-- Verify
SHOW password_encryption;

-- Check which roles still use md5
SELECT rolname, rolpassword LIKE 'md5%' AS uses_md5
FROM pg_authid
WHERE rolcanlogin AND rolpassword IS NOT NULL;
```

### Password Expiration

```sql
-- Set expiration on a role
ALTER ROLE app_user VALID UNTIL '2025-06-01';

-- Check expiration
SELECT rolname, rolvaliduntil
FROM pg_roles
WHERE rolvaliduntil IS NOT NULL;
```

PostgreSQL does not enforce password complexity natively. Use `passwordcheck` module or application-level validation:

```sql
-- In postgresql.conf:
-- shared_preload_libraries = 'passwordcheck'
-- Enforces minimum length and basic complexity
```

### SSL/TLS

```sql
-- Check if SSL is enabled
SHOW ssl;

-- Check current connection's SSL status
SELECT ssl, version, cipher, bits
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();

-- Require SSL for specific connections (in pg_hba.conf):
-- hostssl   mydb   all   0.0.0.0/0   scram-sha-256
-- hostnossl mydb   all   0.0.0.0/0   reject
```

## Security Best Practices

### Principle of Least Privilege

```sql
-- 1. Revoke default public access
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;

-- 2. Create role groups with specific privileges
CREATE ROLE app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_readonly;

CREATE ROLE app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO app_readwrite;

-- 3. Assign users to groups
CREATE ROLE web_app WITH LOGIN PASSWORD '...';
GRANT app_readwrite TO web_app;

CREATE ROLE analyst WITH LOGIN PASSWORD '...';
GRANT app_readonly TO analyst;
```

### Audit Who Has Access

```sql
-- All role memberships
SELECT
    r.rolname AS group_role,
    m.rolname AS member,
    am.admin_option
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
ORDER BY r.rolname, m.rolname;

-- Roles with superuser
SELECT rolname FROM pg_roles WHERE rolsuper;

-- Roles with BYPASSRLS (can circumvent row-level security)
SELECT rolname FROM pg_roles WHERE rolbypassrls;

-- Roles that can create databases or roles
SELECT rolname, rolcreatedb, rolcreaterole
FROM pg_roles
WHERE rolcreatedb OR rolcreaterole;
```

### Avoid Common Security Mistakes

1. **Never use `trust` authentication** in production (allows passwordless access)
2. **Never run applications as superuser** — create dedicated roles with minimal privileges
3. **Don't store passwords in connection strings** in code — use `.pgpass`, environment variables, or secret managers
4. **Pin `search_path`** in `SECURITY DEFINER` functions
5. **Enable SSL** for all non-local connections (`hostssl` in pg_hba.conf)
6. **Audit superuser access** regularly — minimize the number of superuser roles
7. **Use `scram-sha-256`** — md5 is deprecated (PG18 emits warnings on md5 password creation)
