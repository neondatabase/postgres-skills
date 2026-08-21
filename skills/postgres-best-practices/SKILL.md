---
name: postgres-best-practices
description: Best practices and guidelines for working with Postgres. Covers schema design, indexing strategies, query optimization, migrations, and common pitfalls. Use when writing SQL, designing database schemas, optimizing queries, or setting up a Postgres database.
---

# Postgres Best Practices

Guidelines and best practices for working with Postgres, covering schema design, indexing, query optimization, and common pitfalls.

## Deployment Model Guardrail

Before applying operational guidance, determine whether PostgreSQL is self-managed or provided as a managed service.

- **Portable guidance** — SQL, schema, indexes, query plans, locking, transactions, RLS, vacuum behavior, TOAST, and fillfactor generally apply across deployment models.
- **Provider-dependent guidance** — memory limits, connection limits, extensions, replication, backups, recovery, and observability may be configured or exposed differently by each provider.
- **Self-managed guidance** — filesystem layout, PGDATA, tablespaces, WAL archiving, checkpoints, `fsync`, and server-start settings should only be changed when the user controls the PostgreSQL server.

Explain low-level concepts when they help diagnosis, but do not recommend configuration, filesystem, backup, or recovery changes until you confirm the deployment permits them. For managed PostgreSQL, use the provider's skill and current documentation when available; provider-specific guidance takes precedence over this skill.

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
