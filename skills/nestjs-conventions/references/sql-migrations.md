# SQL and Migrations

## SQL contracts and style

- Inspect the DB dialect, schema, naming strategy, and adopted formatter. Preserve existing identifiers and migration formatting.
- Parameterize values. Select dynamic identifiers and sort expressions from controlled mappings; do not interpolate request text into SQL.
- Define identifier, decimal, bigint, timestamp, JSON, and null conversions at the persistence boundary. Do not silently round exact numeric values into JavaScript `number` or change timezone semantics.
- Apply tenant, authorization-related scope, and soft-delete predicates in every relevant query path, including raw SQL and projections.
- Use stable ordering with a unique tie-breaker for pagination. Page parent records before expanding to-many data; keep filtering and total/count semantics consistent.
- For handwritten DDL without another rule, put one column per line, use four spaces, and align column names, types, and attributes within each table. Break long conditions by logical unit.
- Name new DB objects consistently; without an existing policy, use `snake_case` with `pk_`, `fk_`, `uq_`, `ck_`, and `idx_` for primary keys, foreign keys, uniqueness, checks, and ordinary indexes. Respect identifier length limits.
- Add constraints and indexes for concrete invariants and query paths. An application precheck does not replace a DB uniqueness constraint under concurrent writes.

## Migration ownership

Use the project's adopted migration tool, discovery paths, ordering, and deployment command. Using Kysely for reads does not require moving schema ownership to a second migrator.

- Add subsequent changes in new migration files. Do not rename, rewrite, reformat, or append changes to already-applied files.
- Inspect generated SQL before execution. Check data loss, casts, defaults, nullability, foreign keys, index size, locks, and expected duration.
- Keep historical migrations executable without current application entities or Services. Prefer migration-local SQL and stable data definitions.
- Validate both clean installation and incremental upgrade from the supported previous schema when changing migrations.
- Separate large data backfills from long blocking DDL where needed. Keep old and new application versions compatible during rolling deployment through staged expand/backfill/contract changes.
- Do not use schema synchronization or destructive schema recreation as a production migration strategy.
- Execute migrations through the established deployment owner. Do not add competing startup migration runners to every application replica.
- Define failure recovery. A `down` migration must be meaningful; do not describe destructive changes as reversible when data cannot be recovered.

## MikroORM migrations

Apply this section only when MikroORM owns migrations.

- Use the installed driver's migration package and configured Migrator extension. Keep TypeScript source and compiled JavaScript discovery paths correct without loading the same migration twice.
- Retain committed schema snapshots when the project's generator uses them. Review generated output together with metadata and naming-strategy changes.
- Check both individual migration transaction settings and the outer `allOrNothing` setting for statements that must run outside a transaction. Account for earlier migrations already committed if the run is no longer atomic.
- Run schema-diff checks against an isolated database using the actual production dialect and extension configuration.

See [MikroORM migrations](https://mikro-orm.io/docs/migrations) for installed-version configuration and execution behavior.

## PostgreSQL concurrent indexes

When a concurrent index build is required, run it outside transaction blocks and examine locks and failure state. With MikroORM, disabling only an individual migration's transaction while keeping an outer all-or-nothing run can still create dependency/lock problems; configure and test the complete run deliberately. Inspect failed builds for invalid indexes before retrying; `IF NOT EXISTS` does not establish that an existing index is valid.

See [PostgreSQL CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html) for concurrency restrictions. Use the deployed PostgreSQL version's documentation before selecting version-specific DDL.
