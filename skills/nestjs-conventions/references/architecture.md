# Architecture and Responsibility Boundaries

## Feature ownership

Choose the business capability that owns the change before adding classes or folders. Keep its transport, operations, rules, and data access together in one feature module. Preserve the project's established architecture and create only responsibilities needed by the task.

Use domain-driven design where business concepts, invariants, and consistency boundaries warrant it. An aggregate defines a consistency boundary; it is not every object reachable through ORM or GraphQL relationships. A Nest module defines dependency wiring and exposure and need not match an aggregate one to one.

## Responsibilities

| Role | Owns | Keep out |
|---|---|---|
| Controller / Resolver | Transport validation, principal extraction, delegation, public response shape | Direct database access, domain mutations, complex business workflows |
| Service | Cohesive feature operations, authorization, domain behavior, persistence coordination | Unrelated responsibilities and another feature's private state |
| UseCase, when useful | A named workflow, execution order, combined results, required atomicity | Transport handling and unrelated workflows |
| Domain model / policy | Business concepts, calculations, state transitions, invariants | Database/network calls, request objects, application services |
| Repository, when useful | Cohesive reusable queries, loading, locking, persistence mapping | Business workflow coordination and calls back to Services |
| QueryService / QueryRepository, when useful | Read contracts, access policy, projections, specialized queries | Business state changes |
| Infrastructure adapter | External protocols, provider-specific data, technical lifecycle | Core business decisions |

## Operations and orchestration

- Start a straightforward feature with Controller/Resolver → Service. Let that Service use its feature's ORM repository or EntityManager directly when this keeps the operation clear.
- Add a custom `Repository` for meaningful shared queries, loading rules, locking, or persistence behavior. Do not wrap every ORM method or generate a repository interface and implementation for each entity.
- Keep business execution out of controllers and resolvers. Delegate a meaningful operation rather than moving individual database calls through forwarding methods.
- Allow a Service to collaborate with another published Service, a policy, a clock, or an external adapter when those dependencies belong to its responsibility. Dependency count or a `Service` suffix alone does not require a UseCase.
- Extract a named UseCase when a workflow has substantial branching, coordination, failure handling, or a distinct business purpose. Omit a UseCase that only forwards identical arguments to one Service.
- Let a UseCase access its own feature's repositories or EntityManager when it owns that operation. Across independent features, call published application contracts and preserve their ownership boundaries.
- Keep workflow order, decisions, and required persistence visible in one place. Avoid chains of UseCases or tiny forwarding Services that make a single operation difficult to follow.
- Use the installed ORM's transaction facilities or an existing project abstraction. Introduce a transaction runner only for a real abstraction boundary; follow [Transactions](transactions.md).
- Keep authorization effective through HTTP, GraphQL, jobs, and consumers. Pass trusted actor, tenant, or system context explicitly where required.

## Domain behavior and persistence

- Put core invariants and state transitions in domain models or cohesive policies. Let the application operation load required facts, invoke the rule, and arrange persistence.
- Express transitions through meaningful operations such as `approve()` or `cancel()`. Do not bypass protected state through generic patches or unrestricted request assignment.
- Use entities and value objects for identity, behavior, and invariants. Use ordinary functions for stateless calculations; a policy does not require a Nest provider or an interface/implementation pair.
- Allow an ORM entity to serve as the domain model when its behavior and ownership fit. Do not create an identical pure model solely to remove mapping metadata.
- Keep simple data operations simple. Do not manufacture aggregates, value objects, factories, or a domain class for every database table.
- Keep domain behavior free of database and network I/O. Load the state required by a rule before evaluating it, and make partially loaded state explicit.
- Respect the persistence mechanism. Explicit-storage code loads, changes, and saves; a Unit of Work tracks managed changes. Avoid redundant `save()` or `persist()` calls on already managed objects.
- Support concurrency-sensitive invariants with database constraints, version checks, conditional writes, or locks as appropriate. An earlier existence check does not remove a write race.
- Keep managed mutation within the owning business operation. Returning an entity does not permit API code or another feature to bypass its rules.

## Reads and commands

Separate reads from changes by intent. Read methods must not change business state; command methods may load, check existence, and lock the state needed for a change. Returning an ID or result does not turn a command into a query.

- Keep simple reads and writes in one feature Service when their responsibilities remain clear.
- Introduce a QueryService or QueryRepository when read models, query complexity, authorization, or independent evolution benefit from separation. Preserve an existing code-level CQRS convention instead of mixing competing patterns.
- When separating, use `Service` / `Repository` for changes and `QueryService` / `QueryRepository` for reads. Create only the components that perform real work.
- Keep command-side loading in the command operation; do not route every SELECT through a QueryService.
- Let SQL reads map directly to useful projections. Entity results are also valid when their loading and public exposure contracts fit.
- Both paths may share one database and tables. Do not add a bus, handler classes, event sourcing, or separate databases solely to separate read and write code.

## Contracts and collaboration

- Reuse a feature's `CreateXDto`, `UpdateXDto`, or GraphQL input in its Service when the meaning matches and the project permits that coupling. Pass trusted actor and tenant context separately from caller-controlled data.
- Define separate application inputs when transport independence, multiple entry points, authorization, or different semantics require them. Keep business validation effective for callers that bypass transport validation.
- Use `Command`, `Query`, `Result`, `Options`, or domain-owned inputs only when they describe an actual contract. Do not generate a conversion stage for each layer or suffix.
- Keep domain types independent of transport DTOs and application commands. Map only the selected values the domain operation needs.
- Collaborate across features through published application contracts, preferably bulk reads for collections. Keep another feature's repositories and internal managed state private.
- Join data within the owning query implementation when that preserves filtering, ordering, and pagination. Do not reconstruct a database join through one Service call per table.
- Give a cross-feature SQL read model an explicit owner and allowed data dependencies. Reporting access does not authorize changes to other features' tables.
- Introduce interfaces for a real provider choice, published contract, or dependency direction. Mock tests alone do not require interfaces for concrete providers.
- Avoid generic `BaseService` frameworks and shared folders that hide business ownership.

Use [Module structure](module-structure.md) for placement and DI, [API contracts](api-contracts.md) for transport boundaries, [Code style](code-style.md) for readable flow, and [Testing](testing.md) for verification boundaries.

## Framework references

- [Nest providers](https://docs.nestjs.com/providers)
- [Nest modules](https://docs.nestjs.com/modules)
- [Nest CQRS](https://docs.nestjs.com/recipes/cqrs)
