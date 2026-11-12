# GraphQL, DGS, and JPA

## Contents

- [Schema model](#schema-model)
- [Three kinds of types](#three-kinds-of-types)
- [Public relationships and aggregate boundaries](#public-relationships-and-aggregate-boundaries)
- [Entity results and session lifetime](#entity-results-and-session-lifetime)
- [Choose a relationship-loading strategy](#choose-a-relationship-loading-strategy)
- [Real batching and pagination](#real-batching-and-pagination)
- [Authorization and request-scoped caching](#authorization-and-request-scoped-caching)
- [Async execution and context](#async-execution-and-context)
- [Verification](#verification)

## Schema model

**Model GraphQL types around business entities and their relationships. When JPA entities represent the business well, use that model as the default for GraphQL design.** Similarity between the schema and entity model is not itself a design defect.

For the same business object and exposure contract, reuse one server GraphQL type across list, detail, and search. Express screen-specific fields with selection sets and fragments, not new response types such as `OrderListItem`, `OrderDetailResponse`, and `OrderCardData` solely because the screens select different fields.

```graphql
query OrderList {
  orders(first: 20) {
    nodes {
      id
      orderNumber
      customer {
        name
      }
    }
  }
}

query OrderDetail($id: ID!) {
  order(id: $id) {
    id
    orderNumber
    customer {
      id
      name
    }
    items {
      productName
      quantity
    }
  }
}
```

Both operations use the same `Order` and `Customer` schema types. The `orders.nodes` wrapper is illustrative; preserve the project's pagination contract rather than introducing this shape automatically.

Introduce a separate type when business meaning, exposure, or structural purpose actually differs: an aggregate/statistical read result, connection/page metadata, mutation payload, or a distinct public representation. Reuse the existing entity type inside wrappers, such as `OrderConnection` or `CreateOrderPayload`. A DDD aggregate does not require a duplicate GraphQL type merely because it is an aggregate.

Do not mechanically expose every persistence field. Exclude password hashes, internal authentication data, and technical fields that are not part of the public contract. Nullability, identifiers, errors, and compatibility are API contracts, not automatic copies of JPA annotations.

## Three kinds of types

| Type location | Example | Decision |
|---|---|---|
| Server SDL | `OrderListItem`, `OrderDetailResponse` | Avoid duplicates caused only by selected-field differences |
| Client operation-generated result | `OrderListQuery`, `OrderDetailQuery` | Valid: each reflects that operation's actual selection |
| Internal server read projection | SQL projection or aggregate DTO | Valid when useful for queries or mapping; it need not become a new SDL type |

Do not remove client-generated result types or useful internal projections in the name of server schema reuse. Conversely, do not turn every query projection into a public schema type.

## Public relationships and aggregate boundaries

Represent meaningful relationships, such as `Order.customer` and `Order.items`. A technical join entity does not have to become a public node: `Account.roles` can be clearer than exposing an intermediate mapping table. If assignment time, expiry, or grantor are important business concepts, a `RoleAssignment` type can be appropriate.

GraphQL traversal is a **read relationship**, not a command permission or a transaction/aggregate boundary. An Order and Customer can be separate aggregates even when `Order.customer` is selectable.

A GraphQL relationship also does not require adding a JPA association. An existing association may be useful; an ID plus a field resolver and published bulk lookup may be better across ownership boundaries. Do not alter domain ownership merely to make resolver property access convenient.

## Entity results and session lifetime

A resolver may return a JPA entity when the runtime mapping and public contract fit. Do not require a state-identical response DTO solely because the object is persisted. Schema design, Java result representation, and DB storage remain separate decisions.

Plan what happens **after the application query returns**. GraphQL field resolution may occur after its transaction/persistence context has closed. A default getter must not unexpectedly initialize an unloaded association or lazy attribute outside its session.

Choose deliberately:

- Load required values inside the query boundary using an appropriate fetch plan.
- Return a projection/materialized value when that better matches the contract.
- Resolve a relationship by stable IDs through an authorized bulk query contract.

Keep identifier access safe for the actual entity/proxy mapping; do not traverse an unloaded parent graph just to build loader keys. Do not open sessions indefinitely or enable Open Session in View merely to hide an unclear fetch boundary. Where the project already uses OSIV, still make required loading, authorization, and query costs explicit rather than relying on it accidentally.

Do not pass managed entities between threads or transactions expecting the same persistence context or dirty checking. A resolver-returned entity is not permission for API or UseCase code to mutate it.

## Choose a relationship-loading strategy

| Relationship and usage | Default consideration |
|---|---|
| Same feature owns a to-one relationship, almost always needed | Fetch join or the project's equivalent fetch plan |
| Optional/expensive related field | Field resolver and bulk lookup when selected |
| To-many relationship causes row multiplication or harms parent pagination | Page parents first; batch children by selected parent IDs |
| Another feature/module owns the related data | Field resolver + request-scoped DataLoader calling its published bulk QueryService contract |

Use a fetch join only when ownership, cardinality, semantics, and access rules support it; do not introduce blanket eager loading. Account for optional relationships so an inner join does not silently remove valid parents. The schema relationship can remain unchanged whichever loading strategy is used.

Field resolvers and DataLoaders are API adapters. They delegate to application query contracts, not directly to repositories. Fetching another feature's internal entity/repository is not justified by avoiding a DTO or a resolver.

## Real batching and pagination

A DataLoader batches keys only if its implementation makes genuinely bulk downstream calls. Do not loop over keys and execute one repository or service query per ID, or hide that loop inside `loadAll`. Deduplicate where appropriate, issue bounded bulk queries, and map results back correctly.

For list-based batch contracts, preserve input-key order and cardinality, including missing results as required by the loader API. For map-based contracts, use the actual requested keys. Define absence, forbidden access, and errors according to the schema/security contract; do not return another key's value when rows are missing or reordered. Empty child collections and missing parent objects are different cases.

For paged parent lists:

1. Apply all parent-selection predicates, including child-dependent filters or ordering, in the parent query/read model.
2. Page the distinct parent set with a stable ordering and correct total/count semantics where provided.
3. Batch-load requested child fields for those parent IDs.

Do not fetch a large to-many join, paginate multiplied rows, then deduplicate or filter in memory. Loading children after pagination must not change which parents should have been on that page.

A relationship with per-parent `first`, cursor, filter, or ordering arguments has its own contract. Use an argument-aware batch strategy or a deliberate query approach that preserves each parent's requested page. Do not cache different argument sets under only the parent ID or silently implement one global limit for all parents. Chunk bulk requests when DB/provider limits require it; batching does not mean an unbounded query.

## Authorization and request-scoped caching

Authorize each relationship through the application's read contracts. Loading the parent does not automatically authorize every related object or sensitive field. Retain actor, tenant, scope, and exposure rules without revealing forbidden-object existence through inconsistent error handling.

Use request-scoped loaders/caches by default. Do not reuse a mutable DataLoader instance or its cached values across users or requests. Keys must distinguish inputs that can change the result, including relevant field arguments; user/tenant context must be fixed for the request or represented safely when the cache spans scopes. Cross-request caching requires a separately designed security and invalidation policy.

Account for mutations or multiple operations sharing a loader lifecycle: clear or prime affected entries only according to the actual consistency contract. Do not let a request-local cache return stale values after a mutation when fresh state is promised.

## Async execution and context

DataLoader does not require creating an executor or using `CompletableFuture.supplyAsync`. Follow the project's DGS/DataLoader version and execution configuration.

If work moves to another thread, deliberately propagate the required request, actor/tenant, tracing, and security context using the established mechanism. Do not assume `SecurityContext` or a thread-bound Spring transaction follows automatically, and do not use the common pool by habit.

Invoke an appropriate proxied application query boundary on the executing thread when DB access requires a transaction. Pass immutable context and stable IDs rather than relying on an inherited persistence session. Use [Transactions](transactions.md) for execution-boundary details.

## Verification

Select relevant cases from [Testing](testing.md): shared SDL type reuse, selected relationship fields, authorization/absence behavior, lazy-loading safety after query completion, bounded query counts, batch key/result mapping, parent pagination, argument-sensitive cache keys, and request isolation.

Test real fetching when claiming N+1 or pagination behavior. An in-process GraphQL test does not automatically exercise HTTP authentication or filters. Do not treat a successful resolver mock as proof of session lifetime or actual batching.

See [GraphQL/DGS/Hibernate sources](source-notes.md#graphql-dgs-and-jpa). This guide is a modeling and implementation convention, not a requirement to mirror the entire database or to expose every JPA association.
