# Transactions and External Effects

## Contents

- [Assumptions](#assumptions)
- [Boundary ownership](#boundary-ownership)
- [REQUIRED composition](#required-composition)
- [Propagation and execution exceptions](#propagation-and-execution-exceptions)
- [Exceptions and rollback](#exceptions-and-rollback)
- [External effects](#external-effects)
- [Post-commit processing](#post-commit-processing)
- [Concurrency and invariants](#concurrency-and-invariants)
- [Verification](#verification)

## Assumptions

The default model is **Spring-managed beans, ordinary synchronous calls through transactional proxies, one local transaction manager, and DB resources participating in that manager**. It assumes thread-bound declarative transactions.

Check the actual configuration before relying on atomicity. Multiple databases/managers, separate threads, and reactive execution do not automatically share this model. Ensure the persistence tool participates in Spring transactions rather than bypassing them with separate connections or manual commits. See [Spring transaction sources](source-notes.md#spring-transactions).

## Boundary ownership

| Location | Convention |
|---|---|
| API | No business transaction boundary |
| UseCase | Declare a transaction when composed Service changes must succeed/fail together |
| Service | Declare required transactions on public, independently callable DB-changing operations |
| QueryService | Consider `readOnly = true` according to persistence behavior and consistency needs |
| Pure calculation/validation | No transaction without a DB-related need |
| Repo | Participate in the caller's boundary; do not invent independent business commits |
| Domain | No transaction annotations or transaction-management responsibility |

**Do not remove a Service transaction merely because a UseCase also has one.** Support both standalone Service calls and composed calls.

Prefer visible method-level boundaries by default. Keep an established class-level read-only declaration with write-method overrides where appropriate. Do not wrap external-only methods in unnecessary DB transactions.

This does not require removing Spring Data's existing repository transaction behavior. Application business boundaries still belong to Services/UseCases, not repository defaults.

## REQUIRED composition

`REQUIRED` joins an existing transaction or starts one when none exists under the assumptions above.

```text
API -> AccountService.registerManager()
       Service starts a transaction -> commit on success

API -> RegisterCareFacilityUseCase.execute()
       UseCase starts a transaction
       -> AccountService.registerManager() joins
       -> CareFacilityService.register() joins
       -> commit together, or roll back for a rollback-triggering failure
```

An annotation on each Service does not mean a separate physical transaction per call. Conversely, **without an outer transaction**, the two Service calls may commit independently. Sequential calls alone do not provide atomicity.

Do not repeat `propagation = REQUIRED` everywhere when using the default.

## Propagation and execution exceptions

| Situation | Required reasoning |
|---|---|
| `REQUIRES_NEW` | Independent transaction/commit; inner work may survive outer rollback |
| Inner read-only/isolation/timeout settings | Do not assume local settings override an existing joined transaction |
| `this.someMethod()` | Default proxy mode does not apply the called method's new transactional boundary |
| `@Async`, new threads, parallel work | Caller thread-bound transaction does not follow automatically |
| Multiple managers/databases | One annotation does not establish distributed atomicity |

Use `REQUIRES_NEW` for a deliberate independent boundary, not as a generic exception workaround. Suspended outer resources plus a new connection affect pool capacity and locking; inspect both.

Do not default to self-injection or ad hoc proxy lookup to fix self-invocation. Put the boundary on an appropriate separate bean or use an already-adopted explicit transaction mechanism. Verify that callers actually cross the intended boundary.

`readOnly = true` is not a universal write-prevention/security mechanism and does not alone guarantee a shared snapshot across multiple reads. Inspect isolation, manager configuration, and actual DB semantics.

## Exceptions and rollback

Without overriding configuration, Spring normally rolls back for `RuntimeException` and `Error`, not checked exceptions. Inspect global defaults and method-level `rollbackFor/noRollbackFor`; never infer the full policy from exception inheritance alone.

If an inner participant marks the transaction rollback-only, catching the exception in the UseCase does not restore it. Commit can still raise `UnexpectedRollbackException`.

- Propagate failures that must fail the whole business operation instead of reporting success.
- Preserve causes and rollback semantics when translating exceptions.
- Model permitted partial failure as explicit results, independent operations, or compensation.
- Define retryable units and idempotency before retrying; do not blindly retry inside a failed transaction.
- Include flush/commit-time constraint failures in the reasoning. A method returning is not necessarily a successful commit.

## External effects

Local DB rollback does not undo HTTP requests, email, push/SMS delivery, or broker publication. Avoid holding DB connections/locks through unnecessary network waits.

Do not move every external call after commit mechanically. First classify whether its result is a **precondition for persistence** or a **post-success effect**.

| External operation | Direction |
|---|---|
| Notification required only after successful registration | Post-commit work; persist a job/event when delivery durability matters |
| Lookup/encryption result needed as DB input | Obtain before the DB transaction when feasible; recheck necessary conditions in the DB step |
| Payment approval or similarly important external side effect | Explicit progress state, idempotency, retry, reconciliation, and compensation as required |

A UseCase containing network work does not automatically need `@Transactional`. It can coordinate short transactional Service steps without an outer transaction. **That workflow is not one atomic DB operation**; define how intermediate failures are recovered.

Moving a network call to another method does not shorten the transaction if it still executes inside the same boundary. Inspect actual execution and resource lifetime, not method names.

## Post-commit processing

`@TransactionalEventListener` defaults to `AFTER_COMMIT`. Without an active publishing transaction, the listener is normally not invoked unless configured otherwise, such as with `fallbackExecution`.

Distinguish timing, execution, and reliability:

- `AFTER_COMMIT` alone does not make execution asynchronous.
- A post-commit failure cannot roll back already-committed originating changes.
- An in-memory event alone does not guarantee delivery or retry after process failure.
- Resources may still be accessible during a synchronous completion callback even though the original transaction has finished. Do not expect further changes joined to it to commit again.

**For DB writes from a synchronous `AFTER_COMMIT` callback, create an actual new transaction.** A common arrangement is a call through a separate bean's `@Transactional(propagation = REQUIRES_NEW)` boundary. A default `REQUIRED` method is not, by itself, a safe way to guarantee a fresh transaction in that callback. Self-invocation does not fix this.

This rule addresses the synchronous completion case. An asynchronous worker with no current transaction can start its own transaction through a proxied `REQUIRED` entry point. Inspect the actual listener/worker contract rather than adding redundant transaction layers everywhere.

When delivery must survive crashes, record an outbox entry or sending job in the same transaction as the business change and dispatch separately. This does not eliminate duplicate delivery; choose idempotency keys, processing records, or tolerated duplicates according to the contract.

Where Spring Modulith is already used, consider `@ApplicationModuleListener` and a configured Event Publication Registry. The annotation combines asynchronous post-commit processing with a separate transaction; publication persistence and retry behavior still depend on actual registry configuration and version. Do not infer durable delivery, automatic retries, or exactly-once effects from an annotation alone.

Long HTTP calls inside the listener's new transaction still occupy resources. When appropriate, let the listener record work and have a separate dispatcher perform delivery. Do not introduce Modulith or a broker solely to satisfy this section.

See [event and synchronization sources](source-notes.md#events-and-modules).

## Concurrency and invariants

A domain check covers the state it observed, not every competing request. Neither `existsBy...` nor `@Transactional` alone prevents duplicates or conflicting updates. Inspect isolation, constraints, and lock behavior for the actual DB.

- Use DB constraints for enforceable uniqueness and referential integrity.
- For competing changes, select appropriate version checks, atomic conditional updates, or locks. Do not lock every read.
- Verify affected-row counts for version/state-conditional updates. Translate no change, conflict, and missing targets according to the business contract rather than reporting success for an unexpected zero-row update.
- A single-row lock may not protect a multirow invariant or the absence of a not-yet-created row. Analyze the full race and the DB guarantee.

Keep SQL/lock implementation in repositories and business transaction boundaries in Services/UseCases. Atomic SQL is not permission to bypass domain rules; domain invariants and DB predicates/constraints must agree. See [persistence and concurrency sources](source-notes.md#persistence-and-concurrency).

## Verification

Use [Testing](testing.md) to choose relevant checks, not to generate every possible test:

- A standalone proxied Service call commits or fails as specified.
- A later Service failure rolls back earlier changes in an atomic UseCase.
- Actual propagation, rollback policy, and commit-time errors match expectations.
- Success-only follow-up effects do not occur before commit or after rollback.
- Post-commit DB work commits independently where required.
- Concurrent requests, duplicate events, and retries preserve the required invariants.

Mocks cannot prove real proxies, SQL, commits, or rollback. Ensure the test's own transaction does not supply a missing application boundary, and know which transaction observes the verification state. Flush and commit are different observations.

Boundary placement is this skill's convention; Spring does not require a particular package structure or Service/UseCase class name.
