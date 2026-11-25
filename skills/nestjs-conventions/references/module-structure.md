# Module Structure and Dependency Injection

## Feature layout

Organize by feature first and keep a small feature shallow. Preserve an established project layout; otherwise start with the following files and add only those needed.

| Location within a feature | Content |
|---|---|
| `account.module.ts` | Feature registration and public provider exports |
| `account.controller.ts` or `account.resolver.ts` | Transport entry points |
| `account.service.ts` | Cohesive feature operations |
| `dto/create-account.dto.ts` | A REST input DTO; use `*.input.ts` for GraphQL inputs |
| `account.entity.ts` or `entities/` | Entities when used; group several related entities only when useful |
| `account.repository.ts`, when needed | Meaningful custom persistence behavior |
| `account-query.service.ts`, when needed | Distinct read contracts |
| `register-account.use-case.ts`, when needed | A substantial named workflow |

- Keep each class near the feature that owns it. Do not create a folder or a provider merely to complete the table.
- Keep public DTOs in their owning feature's `dto/` or adopted schema location. Keep a small internal-only type beside its consumer when it has no independent contract.
- Use `Repository`, `Dto`, `Input`, and other role suffixes consistently with [Code style](code-style.md). Preserve existing names during unrelated changes.
- Add `queries/`, `use-cases/`, `policies/`, or adapter directories when the amount of related code makes navigation clearer.
- For a feature with substantial responsibilities, separate `api/`, `application/`, `domain/`, `persistence/`, and `infrastructure/` as needed. Keep existing layered features consistent; do not create empty layers or flatten them incidentally.
- In a layered feature, place controllers/resolvers in `api`, operations in `application`, business rules in `domain`, database implementation in `persistence`, and external integrations in `infrastructure`.
- Keep genuinely shared technical capabilities in the project's shared location. A business feature does not become shared infrastructure because several features call it.
- Keep generated files, migrations, and schema resources in their configured tool locations.

Colocate unit tests with their subjects and group integration/E2E tests under `test/`. Follow [Test locations and filenames](testing.md#test-locations-and-filenames) for paths, suffixes, shared support, and discovery; keep test code out of production imports and build output.

## Nest registration

- Register HTTP controllers in `controllers`. Register Services, resolvers, and adapters in `providers` as needed.
- Prefer ordinary constructor injection of concrete providers. Use TypeScript parameter properties such as `constructor(private readonly accountsService: AccountsService) {}` when appropriate.
- Import the module that exports a required provider. Importing its TypeScript class alone does not make it injectable in the consuming module.
- Export deliberate application contracts and shared technical capabilities. Do not export all repositories to make injection errors disappear.
- Reuse imported providers instead of declaring their classes again in each consuming module; repeated registration can create separate instances.
- Keep entities, value objects, and pure functions outside the provider container unless they require its lifecycle or dependencies.
- Keep dependency selection and factories in module configuration. Do not use `ModuleRef.get()` or a general service locator to hide ordinary dependencies.

A module's `exports` controls Nest injection, not arbitrary TypeScript imports. Honor static import boundaries and architecture checks as well as runtime registration.

## Runtime tokens

Use class tokens for ordinary provider injection. Introduce an interface and a custom token for an actual contract or implementation choice. TypeScript interfaces and type aliases are erased and cannot serve as runtime DI tokens.

- Use a class, an abstract class imported as a runtime value, or an exported string/symbol token. Inject a compile-time contract with `@Inject(TOKEN)` and register the same token.
- Keep tokens near their contract or in a small token file, not in a module file that imports their consumers.
- Reuse the same symbol value; two calls to `Symbol('FILE_STORAGE')` produce different tokens.
- Use `useExisting` to alias an already registered provider. Use `useClass` when the token should construct its implementation; do not register both and assume they identify the same singleton.

```typescript
import { Module } from '@nestjs/common';
import { FILE_STORAGE } from './file-storage';
import { S3FileStorage } from './s3-file-storage';

@Module({
  providers: [
    S3FileStorage,
    { provide: FILE_STORAGE, useExisting: S3FileStorage },
  ],
  exports: [FILE_STORAGE],
})
export class StorageModule {}
```

Import runtime tokens and metadata-bearing classes as values. Reserve `import type` for compile-time-only contracts. Check the installed decorator, compiler, and transpiler setup before changing imports or build options.

## Scope and lifecycle

- Default stateless Services to singleton scope. Keep actor, tenant, request, and transaction state in explicit inputs or the project's isolated context, never in mutable singleton fields.
- Introduce request scope for an actual lifetime requirement. It propagates to dependent providers and can expand the per-request object graph.
- Keep request-scoped loaders separate from reusable stateless query capabilities. An ORM RequestContext does not require every Nest provider to be request-scoped.
- Use `@Global()` only for deliberately global infrastructure. Make feature dependencies visible through module imports.
- Investigate provider and import cycles through ownership and dependency direction. Use `forwardRef()` only for a justified remaining cycle and verify its behavior.
- Prefer direct imports for modules and providers. Avoid importing a feature's own barrel from inside that feature and creating hidden cycles.
- Keep constructors limited to dependency assignment. Use supported lifecycle hooks for resources and close pools, consumers, and other handles during shutdown.

## Configuration and entry points

Group configuration through the project's adopted configuration provider and validate required values at startup. Preserve deployed environment-variable names and library-owned keys.

Keep provider credentials and wire formats inside adapters. Put message decoding, acknowledgement, retries, and scheduling at technical entry points; delegate business execution to the owning Service or UseCase. Give each independent job the ORM and authorization context its operation requires.

## Framework references

- [Nest modules](https://docs.nestjs.com/modules)
- [Custom providers and aliases](https://docs.nestjs.com/fundamentals/custom-providers)
- [Injection scopes](https://docs.nestjs.com/fundamentals/injection-scopes)
- [Circular dependencies](https://docs.nestjs.com/fundamentals/circular-dependency)
- [Type-only imports](https://www.typescriptlang.org/docs/handbook/modules/reference#type-only-imports-and-exports)
