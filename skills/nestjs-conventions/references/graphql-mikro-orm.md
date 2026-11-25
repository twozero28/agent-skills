# GraphQL and MikroORM

Apply this reference when GraphQL resolvers return MikroORM entities or use ORM relationships. Also apply [GraphQL](graphql.md) and [MikroORM](mikro-orm.md).

## Public model and backing values

- Model GraphQL around business concepts and meaningful relationships. Reuse one server type for the same concept and exposure contract across list, detail, and search.
- Return an entity when its loaded values and runtime mapping satisfy the public contract. Do not require a state-identical response DTO solely because it is persisted.
- Keep schema design, resolver backing values, and storage mapping separate decisions. A resolver's backing entity need not be the class that declares the GraphQL type.
- When combining entity and GraphQL decorators, verify explicit field exposure, runtime type metadata, nullability, and relation resolution. Do not infer safety from matching property names.
- Preserve an established schema-first or code-first approach. Do not add a second schema-generation strategy.
- Add distinct schema types for different business meanings, exposure contracts, query aggregates, pagination, or operation payloads; reuse existing business types inside wrappers.
- Keep client operation-generated types and useful internal SQL projections. Their different selected fields do not require duplicate server schema types.
- Keep secrets and internal fields outside the GraphQL contract. ORM `hidden` metadata and REST serialization settings do not replace GraphQL field exposure or authorization.

See [Nest resolvers](https://docs.nestjs.com/graphql/resolvers) and [ORM serialization](https://mikro-orm.io/docs/serializing).

## Relationship ownership

- Treat GraphQL traversal as a read relationship, not an aggregate, write-permission, or transaction boundary.
- Expose business relationships without automatically exposing technical join entities. Use a relationship object when its own attributes have business meaning.
- Keep cross-feature relationships behind the owning feature's published application query contracts. An ID plus a field resolver can express a relationship without a cross-feature ORM association.
- Delegate relationship reads to the owning Service or QueryService. Those methods may use ORM loading directly or a Repository when the query merits extraction; do not generate a fixed chain of forwarding layers.
- Treat `Reference.load()` and `Collection.load()` as database work. Keep authorization-aware relationship loading in the owning application read method, including when ORM batching supplies the implementation.
- Keep API code from accessing another feature's EntityManager, repositories, or managed internals to avoid a mapping step.

## Runtime loading and wrappers

- Return a resolved public object or list for relation fields. Do not expose `Ref` or `Collection` wrappers as if they were already the declared GraphQL values.
- Verify what happens after the root resolver returns: child fields may execute later and require their own authorized loading path.
- Keep the required request context available throughout field resolution. Test context setup through the actual GraphQL transport and HTTP adapter.
- Use safe identifiers for relationship lookup; do not traverse unloaded state to construct loader keys.
- Resolve absent, forbidden, and unavailable relationships according to the schema's nullability and error contract.
- Treat populated data as a loading optimization, not proof of authorization. Apply the same access rules whether a relation was preloaded or fetched on demand.
- For subscriptions and deferred work, pass stable values and establish a bounded ORM context for each independent operation.

## Select a loading strategy

| Relationship | Consider |
|---|---|
| Same-feature, frequently required to-one | An explicit `populate`/fetch plan |
| Optional or expensive same-feature relation | Query contract using supported ORM batching or a bulk query |
| Cross-feature relationship | Published bulk read method with a request-local DataLoader |
| To-many relationship affecting parent pagination | Page parents correctly, then load requested children |
| Field arguments define a per-parent page | An argument-aware query and cache strategy |

- Avoid blanket eager loading. Inspect actual SQL for joins, row multiplication, filters, and ordering.
- Keep parent-selection predicates in the parent query, including predicates based on children. Loading children after pagination must not change which parents belong on the page.
- Preserve stable ordering, distinct parent counts, and each parent's pagination contract. Do not apply one global child limit to all parents.
- Use version-supported per-parent loading only after verifying the installed API and generated query; otherwise use an explicit query strategy.

## Built-in batching

- MikroORM supports batching through `Reference.load()` and `Collection.load()` in supported versions. Enable it explicitly through configuration or load options; do not assume it is on by default.
- Check the installed dependency requirements; MikroORM v7 requires the `dataloader` package when using this facility.
- Schedule compatible loads so they can be batched. Avoid sequential per-key awaits that recreate one-query-per-parent behavior.
- Keep ORM batching behind the owning application read method. Preserve the same access checks for batched, preloaded, and individually requested values.
- Use a separate DataLoader when ownership, data source, authorization, or field-argument semantics require a different bulk contract.
- Ensure custom loaders perform real bulk work, preserve key/result correspondence, handle missing values, and bound request sizes.

See [MikroORM dataloaders](https://mikro-orm.io/docs/dataloaders).

## Authorization and cache lifecycle

- Isolate loader caches by request or equivalent operation scope. Do not share mutable loader instances across unrelated users or tenants.
- Include result-changing field arguments in cache keys. Keep actor/tenant context fixed for the cache lifetime or distinguish it safely in keys.
- Do not treat the ORM Identity Map as an authorization cache.
- Clear or prime affected loader entries after successful mutations according to the promised consistency contract. Do not publish uncommitted state to external caches.
- Do not reuse a cached value loaded with one exposure contract for a more privileged or differently filtered field.

## Verification

- Execute actual GraphQL operations for reused types, selected relationships, nullability, denied access, and wrapper conversion.
- Use a real database to verify bounded query counts, loader mapping, parent/per-parent pagination, and transaction visibility.
- Test two independent requests or tenants to verify context and cache isolation.
- Test a mutation followed by a related read when the operation promises fresh state.
- Include a path where the relation was not preloaded; a resolver mock cannot prove loading or context lifetime.
