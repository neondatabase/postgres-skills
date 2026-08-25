---
name: postgres-best-practices
description: Best practices and guidelines for working with Postgres. Covers schema design, indexing strategies, query optimization, migrations, and common pitfalls. Use when writing SQL, designing database schemas, optimizing queries, or setting up a Postgres database.
---

# Postgres Best Practices

Guidelines and best practices for working with Postgres, covering schema design, indexing, query optimization, and common pitfalls.

## Supported Versions

This skill covers PostgreSQL 14 through 18. Version-specific features are tagged (e.g., `[PG15+]`, `[PG18+]`); environment-dependent examples identify required privileges, extensions, or multi-node setup.

PostgreSQL provides 5 years of support per major version. Always run the latest minor release.

| Version | Initial Release    | End of Life        |
| ------- | ------------------ | ------------------ |
| 18      | September 2025     | November 2030      |
| 17      | September 2024     | November 2029      |
| 16      | September 2023     | November 2028      |
| 15      | October 2022       | November 2027      |
| 14      | September 2021     | November 2026      |

Source: [postgresql.org/support/versioning](https://www.postgresql.org/support/versioning/)

## References

| Area                    | Resource                                | When to Use                                                        |
| ----------------------- | --------------------------------------- | ------------------------------------------------------------------ |
| Schema Design           | `references/schema-design.md`           | Designing tables, choosing data types, normalizing, partitioning   |
| Indexing                | `references/indexing.md`                | Choosing index types, composite indexes, partial/covering indexes  |
| Query Optimization      | `references/query-optimization.md`      | Reading EXPLAIN ANALYZE, fixing bottlenecks, planner tuning        |
| Query Patterns          | `references/query-patterns.md`          | CTEs, window functions, lateral joins, UPSERT, JSONB, anti-patterns|
| Performance Diagnostics | `references/performance-diagnostics.md` | pg_stat views, lock analysis, VACUUM, connection management        |
| Logical Replication     | `references/logical-replication.md`     | Pub/sub replication, live migrations, CDC                          |
| Hot Standby             | `references/hot-standby.md`             | Streaming replication, read replicas, failover                     |
| Transaction Isolation   | `references/transaction-isolation.md`   | Isolation levels, lost updates, serialization failures, retry logic |
| Backup & Restore        | `references/backup-restore.md`          | pg_dump/pg_restore, pg_basebackup, PITR, recovery                 |
| Security & Roles        | `references/security-roles.md`          | Privileges, RLS, pg_hba.conf, authentication, SSL                 |
| Bulk Data Loading       | `references/bulk-loading.md`            | COPY patterns, ETL staging, optimizing large loads, batch ops      |
| Connection Pooling      | `references/connection-pooling.md`      | PgBouncer config, pool modes, prepared statements, sizing          |
| Major Version Upgrades  | `references/major-version-upgrades.md`  | pg_upgrade, logical replication migration, pre/post checklists     |
