# TypeScript Style and Readability

## Contents

- [Formatting](#formatting)
- [Vertical reading and semantic grouping](#vertical-reading-and-semantic-grouping)
- [Expressions and control flow](#expressions-and-control-flow)
- [Functions and class layout](#functions-and-class-layout)
- [Naming and contracts](#naming-and-contracts)
- [TypeScript idioms](#typescript-idioms)
- [Asynchronous work](#asynchronous-work)
- [Review](#review)

## Formatting

- Follow the repository's TypeScript, ESLint, Prettier, module-system, and decorator settings. Do not install or reconfigure a formatter during an unrelated change.
- Without an adopted rule, use two-space indentation, braces for control-flow bodies, consistent semicolons, and single-quoted strings. Let the formatter handle wrapping and trailing commas.
- Separate imports from declarations with one blank line. Preserve intentional side-effect import order and follow the configured import sorter.
- Use ordinary indentation, not spaces that align variable names, property colons, assignments, or arguments into columns.
- Keep decorators immediately above their declaration. Do not reorder decorators solely for appearance when composition may affect behavior.
- Work with formatter-supported layouts. Do not add `prettier-ignore`, comments, or configuration changes merely to preserve manual line breaks.

Let long parameter lists use the formatter's vertical layout:

```typescript
constructor(
  private readonly qrCodeRepository: QrCodeRepository,
  private readonly notifications: NotificationsService,
) {}
```

## Vertical reading and semantic grouping

- Write the main path in execution order: obtain required facts, check preconditions, perform the business operation, persist when needed, and produce the result. Put each check after the facts it needs; preserve required security and transaction ordering.
- Keep statements that express one step together. Separate a change of purpose with one blank line, not a blank line after every statement.
- Keep a lookup and its absence check adjacent. Keep a computed value close to its first use; do not collect unrelated declarations at the top.
- Prefer multiline objects when several meaningful fields, nesting, or mapping would otherwise crowd a line. Put one property per line at the same indentation level.
- Use the formatter's multiline form for long signatures, calls, destructuring, and type declarations. Do not force short calls such as `findOne({ id })` onto several lines.
- If a formatted expression is still difficult to scan, name an intermediate value or extract a cohesive operation. Do not widen the line limit to accommodate nested work.
- Prefer visible steps over nesting reads, transformations, writes, and response construction inside one expression. Keep short, obvious expressions compact.

Keep a lookup and its check together, then separate response mapping:

```typescript
async findOne(id: string, actor: Actor): Promise<QrCodeSummary> {
  const qrCode = await this.qrCodeRepository.findOne({
    id,
    tenantId: actor.tenantId,
  });
  if (!qrCode) {
    throw new NotFoundException('QR code not found');
  }

  return {
    id: qrCode.id,
    name: qrCode.name,
    targetUrl: qrCode.targetUrl,
  };
}
```

For a write, keep domain behavior and the required persistence step visible before returning. Use the actual persistence mechanism; do not add a redundant `save()` to a managed entity merely to imitate an example.

See [Prettier rationale](https://prettier.io/docs/rationale) for blank-line and object-wrapping behavior. Preserve the project's formatter options; not every manual line break survives formatting.

## Expressions and control flow

- Use guard clauses and early returns when they reduce nesting. Keep normal execution at a shallow indentation level without changing cleanup, transaction, or authorization behavior.
- Use `if` blocks for branching with work or side effects. Reserve ternaries for short value selection; avoid nested ternaries.
- Name a complex condition when the name expresses a business rule. Keep a simple check direct instead of wrapping every comparison in a helper.
- Use meaningful intermediate names for distinct transformations, repeated calculations, and non-obvious arguments. Avoid intermediates that only rename an obvious scalar.
- Use short `map`, `filter`, `find`, and `some` expressions naturally. Split a chain when it mixes responsibilities or requires nested callbacks; use a loop when state, branching, or ordered asynchronous work is clearer.
- Use destructuring when it clarifies local data. Avoid deep destructuring or extracting so many values that their source becomes unclear.
- Use optional chaining and `??` for genuinely optional values. Do not hide a missing required relation or collapse an explicit-null update into an old value.
- Prefer an options object to a long sequence of ambiguous positional arguments. Keep short, unambiguous signatures direct.

Name a meaningful mapping input before handing it off:

```typescript
const recipients = accounts
  .filter((account) => account.notificationsEnabled)
  .map((account) => ({
    accountId: account.id,
    address: account.email,
  }));

await this.notifications.send({
  template: 'qr-code-created',
  recipients,
  variables: {
    qrCodeName: qrCode.name,
    redirectUrl,
  },
});
```

## Functions and class layout

- Place constructor dependencies and lifecycle setup consistently, then public operations, then private helpers. Keep substantial operations above the helpers that explain their details.
- Group related public operations in business order. For straightforward CRUD, use a consistent read/create/update/delete order when the project has no convention; do not reorder unrelated methods in a small change.
- Arrange related helpers in call order where possible. Keep the main business path understandable without opening many one-line forwarding methods.
- Extract a helper for a cohesive step, a reusable rule, or distracting detail. Do not split functions solely to meet an arbitrary line limit.
- Keep orchestration visible: important checks, mutation order, transaction boundaries, and required awaits should not disappear into vaguely named helpers.
- Use comments for non-obvious constraints or reasons. Avoid decorative separators, phase labels that repeat obvious code, and comments that paraphrase the next statement.

## Naming and contracts

Use business meaning followed by a recognizable responsibility. Use PascalCase for classes and types, camelCase for values and functions, and kebab-case filenames with conventional role suffixes. Preserve consistent existing names.

| Role | Example | File |
|---|---|---|
| Feature wiring | `AccountModule` | `account.module.ts` |
| REST entry point | `AccountController` | `account.controller.ts` |
| GraphQL entry point | `AccountResolver` | `account.resolver.ts` |
| Feature behavior | `AccountService` | `account.service.ts` |
| Custom persistence, when useful | `AccountRepository` | `account.repository.ts` |
| REST input | `CreateAccountDto`, `UpdateAccountDto` | `create-account.dto.ts`, `update-account.dto.ts` |
| GraphQL input | `CreateAccountInput` | `create-account.input.ts` |
| Separate read capability, when useful | `AccountQueryService` | `account-query.service.ts` |
| Separate read persistence, when useful | `AccountQueryRepository` | `account-query.repository.ts` |
| Named workflow, when useful | `RegisterAccountUseCase` | `register-account.use-case.ts` |

- Use `Command`, `Query`, `Result`, `View`, `Options`, or domain-specific names when they describe a distinct contract. Retain meaningful domain `Draft`/`Patch` types; do not generate suffix-based conversion chains.
- Use `Dto` for REST DTOs by default and `Input` for GraphQL input classes. Retain existing `Request`/`Response` contracts when established; verify configured schema-plugin filename matching before changing suffixes.
- Use capability verbs such as `approve`, `cancel`, `register`, and `search`. Use `execute` for a class whose name already describes the workflow.
- Make absence behavior consistent across optional lookups and throwing lookups. Distinguish IDs from objects and use readable booleans such as `isActive` or `canApprove`.
- Name implementations by meaningful provider or mechanism, such as `S3FileStorage`. Do not create an interface to justify an `Impl` class, or use vague `Manager`, `Helper`, `Common`, or `Base` names without a real responsibility.
- Keep public contracts near their owner. A small internal type can stay beside its consumer; TypeScript does not require a separate file for every type.

## TypeScript idioms

- Keep useful local inference. Give public application operations clear input and return types, including `Promise<T>` for asynchronous results; do not annotate every obvious local value or callback.
- Use plain objects and type aliases/interfaces for compile-time data contracts. Use classes for Nest providers, runtime DTO metadata, entities, or behavior that benefits from encapsulation.
- Use ordinary functions for pure transformations and stateless policies when a provider lifecycle adds nothing. Do not create static utility classes or getter/setter pairs for plain data.
- Prefer constructor parameter properties for injected dependencies, usually `private readonly`, within the project's supported compiler settings.
- Use discriminated unions and narrowing to represent meaningful states. Use `satisfies` when supported and useful for checking a literal without discarding inference; it does not validate runtime input.
- Prefer `const` and appropriate readonly references. Do not confuse TypeScript `readonly` with deep immutability, runtime freezing, or authorization.
- Treat untrusted values as `unknown` until validated. Do not use `any`, double assertions, non-null assertions, or casts to conceal a missing check.
- Use definite-assignment declarations only for a verified framework hydration path. Keep runtime DI/schema classes as value imports and use `import type` for compile-time-only references.
- Use `Pick`, `Omit`, and `Partial` for suitable compile-time contracts. Do not assume they create runtime validation metadata or correctly define a business update.
- Use object spreads for suitable value construction. Keep domain changes behind meaningful operations; do not spread untrusted DTOs into entities or use broad assignment that changes protected fields.
- Follow the project's `type`/`interface` and enum conventions. Choose literal unions for simple finite states when suitable; use a runtime enum or constant object when integration requires runtime values.

See [TypeScript narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html), [classes](https://www.typescriptlang.org/docs/handbook/2/classes.html), and [type-only imports](https://www.typescriptlang.org/docs/handbook/modules/reference#type-only-imports-and-exports).

## Asynchronous work

- Await or return work owned by the operation. Do not leave rejections unobserved or launch durable business work with `void`.
- Return a promise directly for simple delegation when no local completion-dependent work is needed. Await inside `try`/`catch` or before cleanup when local failure handling or resource lifetime depends on completion.
- Use `for...of` with `await` for ordered work. Do not use `forEach(async ...)` when completion matters.
- Parallelize independent I/O deliberately, with bounds for large collections. Do not use `Promise.all` for dependent writes or assume a shared transaction/ORM context makes concurrent mutation safe.
- Keep actor, tenant, request, managed entity, loader, and transaction state out of mutable singleton fields. Do not retain context-bound objects beyond their lifecycle.
- Translate failures at their owning boundary with useful context. Do not swallow a persistence error and report success, log secrets, or expose internal errors to clients.
- Keep CPU-heavy synchronous work off request paths when it would block other requests. Use the project's worker or job mechanism when justified.

## Review

- Can the operation be read from top to bottom without reconstructing nested work or following unnecessary forwarding helpers?
- Do blank lines separate meaningful phases while keeping related statements together?
- Are long objects, calls, callbacks, and conditions readable after the configured formatter runs?
- Are types and abstractions clarifying a contract rather than duplicating obvious information?
- Are required checks, state transitions, persistence boundaries, and awaits visible?
