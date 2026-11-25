# GraphQL

## Schema model

- Model types around business concepts and useful relationships. Reuse one server type for the same business object and exposure contract across list, detail, and search.
- Express view-specific selections through operations and fragments. Do not create `OrderListItem`, `OrderDetailResponse`, and `OrderCardData` solely because clients select different fields.
- Create separate types for genuinely different public representations, aggregate/statistical read results, pagination metadata, and operation payloads. Reuse existing business object types inside wrappers.
- Keep client operation-generated types and internal SQL projections when useful; neither requires a distinct public server type.
- Treat GraphQL traversal as a read relationship. It does not define aggregate ownership, mutation permissions, or a shared transaction boundary.
- Preserve the project's code-first or schema-first approach, GraphQL driver, pagination contract, scalar choices, and compatibility requirements.

## Runtime types and field exposure

- With code-first, supply the necessary `@ObjectType`, `@InputType`, `@Field`, and resolver metadata, or use the configured CLI plugin. TypeScript declarations alone do not describe the complete schema.
- Explicitly distinguish IDs, integers, floating-point values, dates, list nullability, and nullable elements where reflection cannot express the contract.
- Inspect CLI plugin configuration before relying on omitted decorators: the plugin can add `@Field` automatically to matching classes, including `.entity.ts` files. Use `@HideField()` where applicable and inspect the emitted SDL.
- Keep credentials, internal authentication data, and restricted technical fields out of the public schema. ORM serialization exclusions do not establish GraphQL exclusion.
- Reuse an existing entity or value as a resolver result when its runtime shape, loading state, and exposure contract fit. Do not require an identical response object merely because the source is persisted.
- Keep schema type reuse separate from runtime result representation. Projections and materialized values remain valid when they satisfy field resolution.
- Verify runtime type discrimination for interfaces and unions. If default resolution depends on class instances, provide those instances or an explicit supported resolver for plain values.
- Apply [API contracts](api-contracts.md) to input validation and partial updates. Nullable GraphQL input fields still need an explicit business null policy.

See [Nest resolvers](https://docs.nestjs.com/graphql/resolvers), [CLI plugin](https://docs.nestjs.com/graphql/cli-plugin), and [interfaces](https://docs.nestjs.com/graphql/interfaces).

## Resolver and authorization boundaries

- Keep root and field resolvers as API adapters. Delegate reads to published QueryService contracts and changes to Services or UseCases; do not inject repositories into resolvers or custom API DataLoaders.
- Load another feature's relationship through its published query contract rather than importing internal repositories or adding persistence associations for convenience.
- Use `GqlExecutionContext` when a transport adapter needs GraphQL arguments or context. Do not assume an HTTP request/response shape in a shared guard or filter.
- Retain actor, tenant, and exposure checks in application query contracts. Parent visibility does not authorize every child relationship or sensitive field.
- Check `fieldResolverEnhancers` when relying on guards, interceptors, or filters for `@ResolveField()`: these enhancers do not run at that level by default. Enable only the required behavior and account for repeated field execution.
- Define missing, forbidden, and failed field results consistently with schema nullability and the security contract. Do not reveal forbidden-object existence through inconsistent lookup behavior.
- Keep operation context available through actual field execution. Give subscription events a deliberate context/cache lifecycle; do not retain request caches indefinitely for a connection.

See [Nest GraphQL execution features](https://docs.nestjs.com/graphql/other-features).

## Relationship loading and batching

- Choose eager materialization, a field resolver, or a bulk lookup according to ownership, selection frequency, cardinality, and authorization. Avoid blanket loading of an entire graph.
- Make custom DataLoader batch functions call genuinely bulk application queries. A loop of one query per key remains N+1 even when wrapped in `Promise.all()` or `loadMany()`.
- Return one batch result per input key in the same order, including duplicate and missing keys when present. Reorder database results explicitly; never let missing rows shift results onto another key.
- Distinguish an empty child collection from a missing parent. Return absence or per-key errors according to the application's contract.
- Bound batch sizes for database/provider limits. Avoid sequentially awaiting independent `load()` calls when they are intended to share a batch.
- Create loader instances and caches for the request/operation scope. Never store a user's mutable loader cache in a singleton provider.
- Include arguments that affect the result in cache identity: parent ID, filters, sort order, cursor, page size, and any changing access scope. Keep fixed request actor/tenant context explicit.
- Invalidate or prime affected entries after successful changes when the operation promises fresh reads. Keep failures and rolled-back mutations from installing successful cached values.

See [DataLoader batching and caching](https://github.com/graphql/dataloader).

## Parent and relationship pagination

1. Apply all predicates and ordering that determine parent membership, including child-dependent conditions, before selecting the parent page.
2. Page distinct parents using a stable order with an unambiguous tie-breaker; preserve the API's count and cursor semantics.
3. Load the requested relationships for those parents using bounded bulk queries or a verified ORM loading strategy.

- Do not paginate multiplied to-many join rows and then deduplicate or filter them in memory.
- Preserve per-parent limits, cursors, filters, and ordering. A single global child limit does not implement each parent's requested page.
- Keep differently parameterized relationship results separate. An argument-aware key alone is insufficient unless the bulk query also implements those arguments correctly.
- Reuse the existing pagination shape. Introduce connection/edge wrappers only when they serve the API contract.

See [GraphQL pagination](https://graphql.org/learn/pagination/).

## MikroORM integration

Read [GraphQL and MikroORM](graphql-mikro-orm.md) when results contain MikroORM entities or relations. Apply its context lifetime, wrapper conversion, population, built-in loader, and mutation consistency rules alongside these schema and authorization boundaries.

## Verification

- Execute representative operations against the generated/runtime schema; inspect shared types, nullability, scalar behavior, and sensitive-field exclusion.
- Verify selected relationship fields, missing/forbidden objects, reordered batch results, duplicate keys, and request/tenant isolation.
- Measure real query behavior when claiming batching or N+1 prevention. Mocked resolvers do not establish database query counts.
- Check parent page membership, stable ordering, per-parent child pagination, and argument-sensitive cache behavior.
- Test mutation-followed-by-read consistency and the actual HTTP authentication path separately from schema-only execution.
- Use the production schema/compiler configuration in schema tests; a test runner that omits the GraphQL CLI plugin can generate different metadata.
