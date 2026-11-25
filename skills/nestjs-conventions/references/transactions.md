# Transactions and External Effects

Use the project's database and transaction mechanisms. Nest dependency injection alone does not define transaction boundaries or propagation.

## Ownership

- Let the Service or UseCase that owns a business operation define its required transaction boundary.
- Put a wider transaction around composed changes when they must succeed or fail together. The workflow may live in a Service or a dedicated UseCase; do not add a class solely to host a transaction.
- Use the project's ORM transaction API directly in the owning application provider unless an adopted module boundary requires an abstraction. Keep ORM handles out of transport DTOs and public business inputs/outputs.
- Keep repository methods responsible for persistence operations, not independently committing fragments of a larger business workflow.
- Decide the atomic unit from business consistency, not from controller routes, resolver nesting, or the number of tables touched.
- Use one local transaction only for operations that actually share its database connection/context. Multiple databases or independently managed clients need a separate consistency design.

## Transaction API and composition

- Prefer the existing ORM callback/decorator mechanism. With MikroORM, use `em.transactional()` or the supported `@Transactional()` integration; select propagation deliberately.
- Introduce an execution-only runner such as `run<T>(work: () => Promise<T>): Promise<T>` only for an actual shared boundary or provider-independent contract. Keep its implementation in the database integration and avoid adding query or entity-access methods to it.
- Define whether a nested call joins an existing transaction or starts one. Prefer joining for an atomic composed operation when the mechanism supports it; do not rely on unspecified defaults.
- Ensure every participating Service and repository resolves the same transaction context. Wrapping a callback is insufficient if its collaborators keep using unrelated clients or forks.
- Within a transaction callback, use its EntityManager or collaborators verified to resolve the same contextual manager. Keep transaction-bound clients local to that execution.
- If using decorators, verify the installed import path, async method requirements, and EntityManager/context lookup. If using an adapter, register it through Nest DI.
- Await each participating operation and the transaction call itself. Do not start unawaited work that outlives the transaction.
- Keep nested Service boundaries compatible with the outer workflow. Do not silently create independent commits inside an atomic operation.

## MikroORM propagation and flush

Apply this section only when MikroORM is used.

- Check installed-version APIs and defaults before configuring propagation. In MikroORM v7, `@Transactional()` defaults to `REQUIRED`, while `em.transactional()` defaults to `NESTED`.
- Use the implicit flush transaction when it covers the entire required unit of queued ORM writes. Use an explicit transaction when read/lock decisions, multiple flushes, or ORM and direct SQL operations must share one boundary; a final flush does not retroactively include earlier reads.
- Select `REQUIRED` explicitly where supported when operations should join one transaction. A savepoint and an independent transaction have different failure and commit behavior.
- Reserve nested savepoints and independent transactions for a demonstrated recovery or isolation requirement. Document the resulting business behavior near the implementation.
- Begin from the current contextual EntityManager. Use the callback's manager or repositories that resolve its transaction context; do not assume a plain independent fork joins it.
- Distinguish a request context, a Unit of Work, a flush, and a committed transaction. Request isolation does not provide atomicity.
- Account for the whole managed Unit of Work when choosing flush points. Do not make each repository flush as though it owned the complete business operation.
- MikroORM transaction callbacks/decorators flush before completion. An inner flush does not mean an enclosing transaction has committed.
- When mixing ORM and SQL queries, keep all participating work on the same transaction and make pending-state visibility explicit. Follow [MikroORM](mikro-orm.md#sql-queries-and-kysely).

See [MikroORM transactions](https://mikro-orm.io/docs/transactions).

## Failure and retries

- Let failures that must abort the business operation escape the transaction boundary. Do not turn them into success values inside the callback.
- Handle recoverable failures only with a supported recovery mechanism. Do not assume catching an exception restores transaction usability or undoes partial in-memory changes.
- Treat rollback as a database operation, not restoration of JavaScript objects. Reload state in a fresh Unit of Work before retrying.
- Retry only identified transient failures, with a bounded policy and an idempotent operation. Re-run the complete intended transaction when its reads and decisions must be repeated.
- Use database constraints for concurrency-sensitive uniqueness; add locks, versions, or conditional writes when the invariant requires them.
- Keep a consistent locking order and check the affected-row result of conditional updates.

## Async execution and concurrency

- Carry only the necessary immutable actor, tenant, tracing, and correlation context into independent work. Establish its own ORM and transaction scope.
- Do not assume an async context crosses a worker, message queue, process, or independently scheduled execution.
- Do not share mutable managed entities across independent concurrent transactions. With MikroORM, use isolated forks or supported clearing options as appropriate.
- Keep ordered dependent writes sequential. `Promise.all` is not a guarantee of useful database parallelism on one transactional connection.
- Keep transactions short; avoid waiting for user input, external HTTP, or unrelated computation while holding database locks.

## External effects and commit

- Treat database changes, HTTP calls, messages, files, and caches as separate resources. DB rollback cannot undo an effect already accepted elsewhere.
- Use idempotency keys and explicit retry/reconciliation behavior for external operations that may be repeated.
- Persist an outbox entry in the same transaction when reliable post-commit delivery is required. Make dispatch and consumers tolerant of redelivery.
- Use an in-memory post-commit callback only when loss between commit and callback is acceptable or another recovery path exists.
- Run required post-commit DB changes in a valid new transaction/context. Do not reuse a completed transaction-bound client.
- Invalidate or publish caches only when the relevant commit succeeds. Coordinate request-local loader state with the API's consistency contract.
- Distinguish a notification callback failing after commit from the business transaction failing; do not report that committed state was rolled back.

## Verification

- Verify each supported standalone operation and, where present, its composition under the owning workflow transaction.
- Induce a failure after an earlier write and confirm rollback through a separate committed read context.
- When mixing ORM and a query builder, verify that both writes roll back together; sharing a database URL is insufficient evidence.
- Verify retry behavior, relevant concurrency conflicts, and external delivery idempotency where changed.
- Keep the actual transaction integration and participating persistence calls real in tests that claim to prove database atomicity.
