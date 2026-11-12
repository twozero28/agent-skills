# Architecture and Responsibility Boundaries

## Contents

- [Design basis](#design-basis)
- [Role contracts](#role-contracts)
- [Service boundaries](#service-boundaries)
- [When to introduce a UseCase](#when-to-introduce-a-usecase)
- [Service collaboration](#service-collaboration)
- [Domain ownership](#domain-ownership)
- [Contracts and code-level CQRS](#contracts-and-code-level-cqrs)
- [Spring Data repository composition](#spring-data-repository-composition)
- [Joined reads and module boundaries](#joined-reads-and-module-boundaries)
- [Calls versus type dependencies](#calls-versus-type-dependencies)

## Design basis

Organize related behavior by feature. Use domain concepts, invariants, and aggregate boundaries to reason about the business, not tables alone. Distinguish API, UseCase, Service, Domain, and Repository responsibilities without generating a separate class for every role in every feature.

These are production boundaries. Test fixtures, seeders, and postcondition inspection follow [Testing](testing.md). Persistence technology does not determine business ownership, nor does this design promise a cost-free persistence migration.

## Role contracts

| Role | Owns | Does not own |
|---|---|---|
| API | Deserialization, format validation, principal extraction, input/output conversion | Repository access, independent-service orchestration, domain mutation |
| UseCase | Cross-responsibility ordering, conditions, combined results, workflow transaction when needed | Queries, persistence mapping, copied service-internal rules |
| Service | One cohesive write responsibility: prepare data, check access, invoke domain behavior, persist | Duplicated core rules, other features' data mutation, hidden orchestration |
| QueryService | Read input interpretation, authorization/exposure policy, results | Business state changes or command invariants |
| Domain | Business concepts, calculations, transitions, invariants, policies | DB/HTTP access, application calls, Spring transaction management |
| Repo | Command-side loading/checks/locking, storage, mapping, DB consistency mechanisms | Workflow coordination, notifications, calls back into services |
| QueryRepo | Search/list/detail/aggregate queries and suitable read results | Business mutations or unnecessary domain reconstruction |
| Infrastructure | External protocols, messaging, scheduling, technical configuration | Core rules hidden in adapters |

API format checks do not replace invariants that every entry point must enforce. Authorization must remain effective for jobs, events, and other non-HTTP callers; prepare an explicit actor or system context when needed.

## Service boundaries

A Service represents a business capability, not a table. One facility-management Service may update basic details, options, and contacts. Conversely, one table can support distinct responsibilities that deserve separate Services.

Check whether its purpose fits one sentence, its methods share rules and reasons to change, and other features can call it as a meaningful capability. Knowing another feature's repository is a warning to reconsider ownership.

Separate read and write responsibilities into `QueryService` and `Service` by default. Split larger responsibilities by coherent behavior, such as approval or search, not arbitrary method/table counts. Do not create an empty counterpart when only reads or only writes exist. Do not reorganize an established project during an unrelated small fix.

## When to introduce a UseCase

| Situation | Default |
|---|---|
| Rename a facility | API -> Service |
| Update several tables within facility management | API -> Service |
| Create an account and register a facility as one workflow | API -> UseCase -> respective Services |
| Controller owns service ordering, branching, and combined results | Extract a cohesive UseCase |
| Class forwards the same arguments to one Service | Do not add it |
| External approval, DB finalization, and compensation require coordination | Consider a workflow-owning UseCase |

Orchestration responsibility matters more than collaborator count. Coordinating several stages of one Service can justify a UseCase if there is a distinct workflow or transaction boundary.

A controller's growing dependency count is a review signal, not a numeric threshold. Split unrelated endpoint responsibilities when appropriate. Do not move HTTP requests/responses, cookies, or CSRF handling into a UseCase just to reduce controller fields, and do not create one giant dependency-hiding UseCase.

Place a UseCase in the owning feature's `application` or `application/usecase`; avoid ownerless `global/usecase`. Introduce a new feature only for a genuinely separate business responsibility. Do not chain UseCases for reuse by default: extract reusable capabilities into Services and keep orchestration readable in one place. Respect an explicitly adopted project alternative.

A UseCase must not inject or call a Repo, QueryRepo, ORM session, SQL executor, or persistence mapper. This applies to read-only composition too. Use published Service/QueryService contracts.

## Service collaboration

Compose **independent business Services** in a UseCase. A facility Service should not secretly coordinate account creation and notification workflows.

This does not prohibit a Service from using a policy, validator, clock, encryption contract, storage capability, or external lookup needed for its own responsibility. `EncryptionService` may name a limited technical contract, not an independent business workflow. Judge actual responsibility, not the `Service` suffix; renaming a business Service to `Helper` or `Client` does not bypass the boundary.

## Domain ownership

Put core rules, state transitions, and invariants in domain models or policies immediately; do not wait until they become complex or duplicated. Services obtain data and execution context, invoke the rules, and save the result. Supply externally obtained facts to domain behavior rather than letting the domain call repositories or APIs.

Simple data changes without distinct business rules may use Service + Repo directly. Do not create a state-identical domain copy for data transfer, and do not bypass an existing invariant because an operation looks like CRUD. Model approval and cancellation as business actions, such as `approve()` and `cancel()`, rather than a generic status patch.

New pure domain types use business values, not HTTP DTOs or generated SQL records. An existing JPA entity may already be a useful domain model; do not automatically remove its annotations or create a duplicate pure object.

| Operation | Flow for explicit-storage persistence |
|---|---|
| Change an existing domain object | Load -> invoke domain behavior -> save |
| Create a domain object | Prepare Draft/creation values -> construct -> save |
| Simple direct update without separate rules | Service -> Repo update |

Do not load a nonexistent object before creating it. Prerequisite checks remain valid. Respect the persistence tool's actual save/dirty-checking contract; do not add redundant `save()` calls to imitate a diagram.

An aggregate is a business consistency boundary, not every reachable JPA or GraphQL object. Keep related objects together when invariants require it, but do not make the entire relational graph one aggregate. Domain checks and concurrent DB protection must agree; see [Transactions](transactions.md#concurrency-and-invariants).

## Contracts and code-level CQRS

Pass explicit IDs, values, commands, and results between Services. A UseCase must not call setters on a returned entity and rely on dirty checking; request changes through the owning Service.

Reuse domain-owned `Draft/Patch` when input meaning and validation match. Add `Cmd` for a distinct business contract, not a copying stage. See [Naming](naming.md#input-reuse).

Separate reads and writes by **business intent**:

- Command Services may load state, check existence, or lock through their own Repo. `findById`, `existsBy...`, and `findByIdForUpdate` do not require a QueryService detour.
- QueryService/QueryRepo own data-serving list/detail/search/statistics contracts. Retain authorization, filters, and exposure rules. A read-only transaction hint does not enforce this boundary by itself.
- A write may return an ID or operation result. A return value does not make it a query.
- Both paths may use the same database and tables. Do not add separate databases, event sourcing, queues, handlers, or a bus solely for CQRS.
- Prefer complete business operations over splitting a change into public read/write steps that leave consistency to callers.

Choose read results for their purpose. SQL-oriented tools can map directly to read DTOs without reconstructing domain objects. JPA queries can return entities when appropriate, subject to [loading and session boundaries](graphql-jpa.md#entity-results-and-session-lifetime). DTO mapping does not permit bypassing QueryService.

## Spring Data repository composition

Do not declare two independently managed `JpaRepository` beans for the same entity and data source solely to label one read-only. Expose a narrow read contract and use one backing repository when that satisfies the responsibility. Do not add a class that only forwards identical methods.

Conceptual composition:

```java
public interface OrderQueryRepo {
    List<Order> findByStatus(OrderStatus status);
}

public interface OrderRepo extends JpaRepository<Order, UUID>, OrderQueryRepo {
}
```

Each public interface belongs in its own file. In this arrangement, `OrderQueryRepo` is a plain read contract, not a second Spring Data repository; QueryService injects it, and command Service injects `OrderRepo`. Keep the actual declarations in the project's accepted packages. If multiple beans implement the contract, resolve injection explicitly. This is an optional composition example, not a reason to generate interfaces everywhere.

Use derived methods or short `@Query` declarations for fixed queries and projections when only selected fields are needed. For dynamic filters, complex joins, or aggregates, use the project's existing query implementation technique, such as Querydsl and repository fragments. A read contract can expose both derived and custom queries. Preserve the framework's fragment-discovery naming conventions; see [Naming](naming.md#implementation-names).

## Joined reads and module boundaries

Do not call one Service per table and join results in memory merely because SQL would join tables. Avoid repeated queries and incorrect sorting, filtering, or pagination.

1. Within one business owner's data, let its QueryRepo produce the required shape.
2. Across independent features/modules, first use published read contracts, preferably bulk contracts.
3. If a cross-boundary SQL read model is required, explicitly define its owner and allowed data dependencies. Do not treat a report as permission to reach into any module's tables.

Never put SQL in a UseCase or use a read model to modify another feature's data. For GraphQL relationship selection, batch loading, and parent pagination, use [GraphQL and JPA](graphql-jpa.md).

## Calls versus type dependencies

This convention does not require a port at every boundary or globally one-way package compilation dependencies. Stricter project/module rules still take precedence.

- A Service may depend on its own concrete Repo.
- QueryRepo may accept an application-owned read input or construct a read DTO without copying it merely to change a suffix. It must not call application execution logic.
- Other features cooperate through published application contracts, not internal repositories or persistence entities. Java `public` alone does not establish a module contract.
- If a repository interface is useful, an application-owned capability contract or domain-owned aggregate storage contract can be appropriate. Choose by purpose; do not create both for the same role. Put its DB implementation in `persistence`.
- Investigate feature cycles by revisiting ownership and contracts. Do not hide them with `@Lazy`, global common packages, or indiscriminate events.

Avoid per-table layer generation, mandatory interface/implementation pairs, generic BaseService/BaseRepository frameworks, and giant utilities. Neither adopting JPA nor replacing it resolves domain modeling by itself.
