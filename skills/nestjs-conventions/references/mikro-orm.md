# MikroORM

Apply this reference only when the project uses MikroORM. Preserve its driver, mapping style, metadata provider, and supported package versions.

## Registration and compatibility

- Inspect the lockfile, `@mikro-orm/nestjs`, driver, decorator mode, and compiled configuration before selecting APIs.
- Register feature-owned repositories through the existing module configuration. Export application contracts rather than another feature's ORM repositories.
- Keep entity discovery consistent between application startup, the CLI, tests, and compiled deployment output. Nest `autoLoadEntities` does not replace CLI entity configuration.
- Match decorator imports to the installed major version. For v7 with legacy decorators, use the supported `@mikro-orm/decorators/legacy` exports and configure the selected metadata provider explicitly.
- Do not change the project's decorator system, module format, or ORM version merely to reuse an example.

See [Nest integration](https://mikro-orm.io/docs/usage-with-nestjs) and the [Nest MikroORM guide](https://docs.nestjs.com/data/mikroorm).

## Context ownership

- Give each independent request or job its own Identity Map. Reuse the established Nest request-context integration.
- Keep singleton providers when contextual EntityManager resolution already supplies isolation. Nest request scope and ORM request context are different mechanisms.
- Register manual context middleware after body/query parsing and before middleware that accesses the ORM. Verify ordering for the actual HTTP adapter and GraphQL driver.
- Establish context at queue, scheduler, CLI, and other non-HTTP entry points. Reuse an existing context for nested application calls; do not create independent contexts in every service.
- Do not enable `allowGlobalContext` to conceal missing production context setup.
- Pass IDs and immutable values across jobs, workers, and independent executions. Reload managed entities in the receiving context.
- Keep request contexts bounded. For long-running streams or subscriptions, establish an appropriate context for each independent database operation instead of retaining one Identity Map indefinitely.
- Treat a request context as entity isolation; it does not open a database transaction.

See [Identity Map and Request Context](https://mikro-orm.io/docs/identity-map).

## Domain and persistence responsibilities

- Let a useful entity own business rules and state transitions. Do not duplicate its state into a second domain model solely because it has ORM mappings.
- Keep domain behavior synchronous and free of database/network I/O. Load the state required by a rule before invoking it.
- Keep EntityManager access, ORM repositories, query construction, and persistence mapping in `persistence`.
- Use Services for loading through repositories, authorization, domain behavior, and persistence coordination. Keep API and UseCase code from mutating managed entities.
- Keep a transaction adapter's EntityManager access limited to transaction/context execution. Do not turn that adapter into a query or entity-access escape hatch.
- Define cascade and orphan-removal rules from lifecycle ownership. A convenient relation is insufficient reason to cascade deletion.

## Managed state and flushing

- Register new entities for persistence. Already-loaded managed entities are tracked without repeated `persist` calls.
- Treat `flush()` as applying the current Unit of Work, which can include changes from multiple repositories. It is not an operation scoped to one entity.
- Align flush ownership with the Service or composed workflow transaction; do not flush automatically in every repository method.
- Flush deliberately when later database work needs generated values or pending changes. Do not equate a flush inside an outer transaction with its final commit.
- Distinguish managed entities, detached values, and SQL projections. Do not assume a plain query result participates in change tracking.
- Do not use `assign`, bulk updates, or direct SQL to bypass domain invariants for ordinary business changes.
- When direct SQL changes managed data, plan synchronization before further entity use. Avoid refreshing away pending changes or clearing an unrelated Unit of Work.

See [Entity Manager](https://mikro-orm.io/docs/entity-manager) and [Transactions](transactions.md).

## Relations and loading

- Represent loading state with the project's supported `Ref`, `Loaded`, and collection APIs. A TypeScript entity annotation alone does not prove initialization.
- Choose a bounded fetch plan using `populate`, a query projection, or a bulk relationship query.
- Keep initialization checks meaningful. Do not use unchecked collection access to disguise an unloaded relationship as an empty result.
- Treat partially loaded collections as partial data; do not base full-membership invariants or replacement operations on them.
- Distinguish removing an association from deleting the related entity.
- Keep GraphQL wrapper exposure and batching rules in [GraphQL and MikroORM](graphql-mikro-orm.md).

See [Type-safe relations](https://mikro-orm.io/docs/type-safe-relations) and [Collections](https://mikro-orm.io/docs/collections).

## SQL queries and Kysely

- Use ORM queries or QueryBuilder where suitable. Adopt Kysely only when the project already uses it or the task justifies it.
- Keep SQL/query-builder clients inside repositories. Return purpose-built projections through QueryService contracts.
- In MikroORM 7.1+ SQL drivers, obtain `getKysely()` from the current transactional EntityManager when queries must participate in that transaction. Verify installed-version support before relying on this behavior.
- Do not cache a transaction-bound Kysely instance in singleton state. An independently created client, pool, or EntityManager fork does not join a transaction merely by targeting the same database.
- Choose raw database names or ORM-aware naming/value conversion deliberately. Do not assume the default Kysely instance applies every ORM mapping, filter, hook, or access rule.
- Apply tenant, authorization, and soft-delete predicates explicitly where the query path requires them.
- Flush pending ORM state before a direct SQL read that must observe it. Plan managed-state consistency after direct writes.
- Verify shared transaction behavior with a real rollback test; verify projection scalar types and nullability against the driver.

See [Kysely integration](https://mikro-orm.io/docs/kysely#transaction-context).

## Verification

- Exercise independent request/job isolation, required relation loading, and the complete changed persistence path.
- Test the relevant standalone Service and composed workflow rollback paths without mocking the database calls under test.
- Keep migration generation, review, and deployment checks in [SQL and migrations](sql-migrations.md).
