# Architecture and Responsibility Boundaries

## Feature ownership

Choose the business capability that owns a change before choosing its classes. Group its API, application behavior, domain rules, persistence, and external adapters together. Create only the roles needed by the work.

Use entities, value objects, aggregates, and policies to express business concepts and invariants. An aggregate is a consistency boundary; it is not every object reachable through ORM or GraphQL relationships. A Nest module defines dependency wiring and exposure, which need not match an aggregate one to one.

Apply these boundaries to new or structurally changed production code. Keep a small fix within its existing scope. Test support follows [Testing](testing.md).

## Role contracts

| Role | Owns | Keep out |
|---|---|---|
| Controller / Resolver | Transport validation, principal extraction, input/output conversion | Repository calls, domain mutation, independent-service orchestration |
| UseCase | Workflow order, conditions, combined results, required atomicity | Repositories, ORM clients, SQL, persistence mapping |
| Service | One cohesive change capability: obtain facts, check access, invoke rules, persist | Other features' internal state, duplicated domain rules, hidden workflows |
| QueryService | Read contracts, authorization, exposure policy | Business state changes |
| Domain | Business concepts, calculations, state transitions, invariants | DB/HTTP calls, application services, request objects |
| Repo | Command-side loading, existence checks, locks, persistence mapping and writes | Workflow coordination or calls back to application services |
| QueryRepo | List, detail, search, aggregate queries and suitable read results | Business mutations or unnecessary domain reconstruction |
| Infrastructure adapter | External protocols, messages, schedules, technical configuration | Core business decisions |

## Execution paths

| Work | Path |
|---|---|
| One change capability | API → Service → Repo |
| Read contract | API → QueryService → QueryRepo |
| Composite workflow | API → UseCase → participating Services |
| Core rule evaluation | Service → domain model or policy |
| Scheduled or consumed work | Scheduler / Consumer → UseCase or Service |

Keep the application boundary even for a short endpoint. Omit a UseCase that only forwards identical arguments to one Service. A UseCase may compose read contracts through QueryServices; it must not inject or call a Repo, QueryRepo, EntityManager, SQL client, or persistence mapper, including for reads.

When a workflow owns a transaction, use the project's configured transaction boundary or a narrow execution-only transaction runner. Do not expose an ORM manager or SQL executor to the UseCase through that runner. See [Transactions](transactions.md).

## Services and orchestration

- Define a Service by one coherent business responsibility, not one table. Updating an account and its contacts can be one capability; registering an account and subscribing it to a separate product can require a UseCase.
- Compose independent business capabilities in the owning feature's UseCase. Keep execution order and failure behavior visible in one place rather than chaining UseCases for reuse.
- Allow a Service to use a policy, clock, encryption contract, storage capability, or external lookup needed by its own responsibility. Judge the collaborator's behavior, not whether its name ends in `Service`.
- Treat growing Controller/Resolver dependencies as a review signal. Extract a meaningful workflow; do not move HTTP response handling or unrelated endpoints into a giant UseCase.
- Keep authorization effective for jobs and message consumers as well as HTTP callers. Pass explicit actor, tenant, or system context where the operation needs it.

## Domain behavior and persistence

- Put core rules in domain models or policies when introducing the behavior. Let Services obtain external facts, invoke those rules, and arrange persistence.
- Express transitions through operations such as `approve()` or `cancel()`. Do not bypass invariants through a generic status patch or unrestricted assignment from a request.
- Allow an ORM entity to serve as the domain model when its behavior and ownership fit. Do not create an identical pure model merely to remove mapping metadata.
- For simple data operations without distinct domain rules, use Service and Repo without inventing a domain class for every table.
- Respect the persistence mechanism: explicit-storage code loads, changes, and saves; a Unit of Work can track managed changes. Do not add redundant `save()` calls to imitate a layer diagram.
- Keep DB constraints, locking, and version checks consistent with domain invariants. A prior existence check alone does not resolve a concurrent write race.
- Pass IDs, values, commands, and results across independent features. A returned entity never authorizes API or UseCase code to mutate it implicitly.

## Code-level CQRS

Separate commands from read contracts by business intent. Commands may load, check existence, and lock through their own Repo; those SELECTs do not require a QueryService detour. Queries may enforce access and exposure rules but must not change business state.

Use `Service` / `Repo` for changes and `QueryService` / `QueryRepo` for reads. Create only the side needed. An ID or operation result returned by a command does not make it a query.

Both paths may share the same database and tables. Do not introduce a bus, event sourcing, separate databases, or handler classes solely to satisfy CQRS. SQL reads may map directly to projections; entity results are also valid when their loading and exposure contracts fit.

## Collaboration and contracts

- Collaborate across features through published application contracts, preferably bulk read contracts for collections. Keep another feature's repositories and internal entities private to its ownership.
- Join data within one owner's QueryRepo when that preserves correct filtering, ordering, and pagination. Do not issue one Service call per table merely to reconstruct a database join in memory.
- Give a cross-feature SQL read model an explicit owner and allowed data dependencies. Reporting requirements do not grant permission to mutate other features' tables.
- Allow a Service to use its own concrete Repo. Introduce interfaces for an actual contract, provider choice, or module boundary; mock tests alone do not require an interface for every class.
- Let a QueryRepo accept a suitable application read input or construct a read result without a duplicate DTO stage. It must not call application execution logic.
- Reuse domain-owned `Draft` / `Patch` inputs when their meaning and validation match. Service and UseCase contracts must not depend on API `Request` / `Input` classes, and domain types must not depend on application commands.
- Avoid generic `BaseService` frameworks, per-table layer generation, and shared folders that hide business ownership. Follow stricter existing module rules when present.

Use [Module structure](module-structure.md) for dependency wiring and [API contracts](api-contracts.md) for transport boundaries.
