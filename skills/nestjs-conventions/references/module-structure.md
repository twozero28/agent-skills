# Module Structure and Dependency Injection

## File ownership

Organize by feature first. Place the feature's `*.module.ts` at its root and add only the classifications that contain real code.

| Location within a feature | Content |
|---|---|
| `api/` | Controllers, resolvers, transport DTOs, API-specific guards/pipes/filters |
| `application/` | Services, QueryServices, UseCases, commands, read contracts, application events |
| `domain/` | Entities, value objects, policies, Draft/Patch inputs, domain facts |
| `persistence/` | Repos, QueryRepos, queries, persistence mapping, DB-specific configuration |
| `infrastructure/` | External clients, provider DTOs, message adapters, schedulers, technical configuration |

Keep DTOs near their owning layer, normally in separate files. Add local `dto`, `query`, or `use-case` directories only when navigation benefits. Keep a simple internal-only type in its consuming file when it has no independent contract.

Keep DB access in `persistence` and other integrations in `infrastructure`. Preserve an established alternative during a small change; do not mix equivalent layouts in newly added code. Classify caches by responsibility rather than product name.

Keep shared capabilities in an existing shared location only when their contract is actually shared. A business feature does not become shared infrastructure because several other features call it. Keep generated files, migrations, and schema resources in their configured tool locations.

## Nest registration

- Register HTTP controllers in `controllers`. Register application providers, resolvers, and adapters in `providers` as needed.
- Import the module that exports a required provider. Importing its TypeScript class alone does not make it injectable in the consuming module.
- Export only intended application contracts or technical capabilities. Do not export every Repo to make injection errors disappear.
- Reuse an imported provider rather than declaring its class again in each consuming module; repeated registration can create separate instances.
- Keep entities and value objects as ordinary objects unless they genuinely need a provider lifecycle. Do not register a class merely because it lives in `domain`.
- Keep provider factories and dependency selection in the composition layer. Do not use `ModuleRef.get()` or a general service locator to hide ordinary application dependencies.

A module's `exports` controls Nest injection, not arbitrary TypeScript imports. Honor the project's static import boundaries and architecture checks as well as runtime registration.

## Runtime tokens

TypeScript interfaces and type aliases cannot be DI tokens. Use an actual class, an abstract class imported as a runtime value, or an explicitly exported string/symbol token. Inject an interface with `@Inject(TOKEN)` and register that same token.

Keep tokens in their owning contract or a small token file, not in a module file that imports their consumers. Reuse the same symbol value; `Symbol('FILE_STORAGE')` created twice produces different tokens.

Use `useExisting` when another token must refer to an already registered provider. Use `useClass` when the token should construct that implementation. Do not accidentally register both and assume they identify the same singleton.

```typescript
export const FILE_STORAGE = Symbol('FILE_STORAGE');

// In the owning module's metadata:
providers: [
  S3FileStorage,
  { provide: FILE_STORAGE, useExisting: S3FileStorage },
],
exports: [FILE_STORAGE],
```

Import runtime DI tokens and classes as values. Reserve `import type` for compile-time-only contracts. Verify emitted metadata and the existing decorator/transpiler setup before changing imports or build options.

## Scope and lifecycle

- Default stateless application Services to the normal singleton scope. Store actor, tenant, request, or transaction state in explicit inputs or the project's isolated context, never in mutable singleton fields.
- Introduce request scope for an actual lifetime requirement. It propagates through dependent providers and can expand the per-request object graph.
- Keep request-scoped loaders separate from reusable stateless query capabilities. An ORM RequestContext is not a reason to mark every Nest provider request-scoped.
- Use `@Global()` only for deliberately global infrastructure. Keep feature dependencies visible through module imports.
- Investigate provider and import cycles by revisiting ownership and extracting the appropriate contract. Use `forwardRef()` only for a justified remaining cycle, with its runtime behavior verified.
- Prefer direct imports for modules and providers. Do not import a feature's own barrel from inside that feature and create a hidden circular import.
- Keep constructor work to dependency assignment. Use the established lifecycle hooks for resources, and close pools, consumers, and other handles during shutdown.

## Configuration and entry points

Group custom configuration by application/service and feature using the project's adopted configuration provider. Validate required values at startup. Preserve deployed environment-variable names and library-owned keys; configuration nesting is not permission to rename an operational contract.

Keep provider credentials and wire formats inside infrastructure adapters. Put message decoding, acknowledgement, retries, and scheduling at technical entry points; delegate business execution to a UseCase or Service. Give each independent job the execution context required by its persistence and authorization contracts.

## Framework references

- [Nest modules](https://docs.nestjs.com/modules)
- [Custom providers and aliases](https://docs.nestjs.com/fundamentals/custom-providers)
- [Injection scopes](https://docs.nestjs.com/fundamentals/injection-scopes)
- [Circular dependencies](https://docs.nestjs.com/fundamentals/circular-dependency)
- [Type-only imports](https://www.typescriptlang.org/docs/handbook/modules/reference#type-only-imports-and-exports)
