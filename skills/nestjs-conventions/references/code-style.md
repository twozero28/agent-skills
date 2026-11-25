# TypeScript Style and Naming

## Formatting and reading order

- Follow the repository's TypeScript, ESLint, Prettier, module-system, and decorator settings. Do not install or reconfigure a formatter during an unrelated change.
- Without an adopted rule, use two-space indentation, braces for control-flow bodies, and consistent semicolons. Let the formatter decide wrapping and trailing commas.
- Separate imports from declarations and group related implementation steps with blank lines. Avoid decorative comments and horizontal alignment tied to name length.
- Group public operations in Read → Create → Update → Delete order when CRUD fits. Keep private helpers below public operations and arrange them in a readable call order.
- Extract a helper when its name reveals a cohesive step. Keep essential execution order visible; avoid dozens of one-line forwarding helpers.
- Use comments for a non-obvious reason or constraint. Put decorators immediately above their declaration and do not reorder them merely for appearance when composition may affect behavior.

## Role names

Use business meaning, then responsibility, then a necessary implementation identity. Preserve consistent existing names instead of renaming unrelated files.

| Role | Class | File |
|---|---|---|
| Feature wiring | `AccountModule` | `account.module.ts` |
| REST entry point | `AccountController` | `account.controller.ts` |
| GraphQL entry point | `AccountResolver` | `account.resolver.ts` |
| Change capability | `AccountService` | `account.service.ts` |
| Read capability | `AccountQueryService` | `account.query-service.ts` |
| Workflow | `RegisterAccountUseCase` | `register-account.use-case.ts` |
| Command persistence | `AccountRepo` | `account.repo.ts` |
| Read persistence | `AccountQueryRepo` | `account.query-repo.ts` |

Use PascalCase for classes/types and camelCase for methods/values; use kebab-case filenames with the role suffix. Retain framework-required names. Do not add `Cmd` to a Service name; it identifies a command input.

Name real implementations by provider or mechanism, such as `S3FileStorage` or `RedisTokenStore`. Do not create interfaces to justify names or add vague `Impl`, `Helper`, `Manager`, `Common`, or `Base` suffixes without a defined responsibility.

Use capability verbs such as `approve`, `cancel`, and `search`; `execute` fits a UseCase whose class already names the workflow. Make absence behavior clear and consistent across optional lookups and throwing lookups. Distinguish IDs from objects and use readable boolean names such as `isActive` or `canApprove`.

## Data contracts

| Owner | Purpose | Default suffix |
|---|---|---|
| API | REST input/output | `Request` / `Response` |
| API | GraphQL schema input | `Input` |
| Application | Change input / read input | `Cmd` / `Query` |
| Application | Read result / operation result | `View` / `Result` |
| Persistence | Distinct search conditions | `Cond` |
| Domain | Creation values / selected property changes | `Draft` / `Patch` |

Do not generate every row or a complete chain of conversions. Reuse a domain Draft/Patch directly as Service input when meaning and validation match; introduce a Cmd for a distinct application contract. Return a scalar ID when it is the whole result.

Keep public DTOs near their owner in separate files. Reserve inline object types for short internal details. Use concrete classes where runtime validation, schema generation, transformation, or metadata needs them; use interfaces/type aliases for compile-time contracts when appropriate.

A Patch need not make every property optional. For optional fields, preserve omission, supplied values, and explicit null through validation and mapping. Follow [API contracts](api-contracts.md); do not use truthiness checks to discard valid `false`, `0`, or empty-string values.

## Type safety and state

- Give exported application operations clear input and return contracts, including `Promise<Result>` for asynchronous results. Keep useful local inference instead of repeating obvious types everywhere.
- Treat untrusted input as `unknown` until checked. Do not use `any`, double assertions, or non-null assertions to conceal a missing validation or invariant.
- Narrow unions and nullable values explicitly. Use definite-assignment declarations only when a known framework hydration path establishes the value, and verify that path.
- Prefer `const` and readonly references where reassignment is unnecessary. Do not mistake TypeScript `readonly` for runtime freezing, validation, or deep immutability.
- Keep domain mutation behind meaningful operations. Avoid broad `Object.assign(entity, input)` and spread-based updates that permit protected fields or erase presence semantics.
- Keep API DTOs out of application/domain inputs and ORM/SQL handles out of business contracts. Use [Module structure](module-structure.md) for runtime token imports.

## Asynchronous control flow

- Await or return work whose completion belongs to the current operation. Do not leave rejected promises unobserved or launch durable business work with `void`.
- Use `for...of` with `await` for ordered operations. Do not use `forEach(async ...)` when the caller must await completion.
- Parallelize independent I/O deliberately, with bounds where needed. Do not use `Promise.all` for dependent writes or assume it makes operations sharing a transaction or mutable ORM context safe.
- Keep request-specific state out of singleton fields. Do not capture a transaction manager, managed entity, or request loader for use after its lifecycle ends.
- Re-throw or translate failures at their owning boundary with useful context. Do not swallow a persistence error and report success, log secrets, or expose internal error objects directly to clients.
- Keep CPU-heavy synchronous work off request paths when it would block concurrent requests; use the project's worker or job mechanism when justified.

## Language references

- [TypeScript narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TypeScript classes and readonly members](https://www.typescriptlang.org/docs/handbook/2/classes.html)
- [Type-only imports](https://www.typescriptlang.org/docs/handbook/modules/reference#type-only-imports-and-exports)
