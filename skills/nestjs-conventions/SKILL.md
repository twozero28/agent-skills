---
name: nestjs-conventions
description: >-
  Apply when implementing, changing, refactoring, testing, or reviewing TypeScript
  NestJS backends. Covers feature modules, dependency injection, domain behavior,
  DTOs, REST/GraphQL contracts, readable top-down code, naming, asynchronous work,
  persistence, transactions, migrations, and tests. Use the MikroORM references
  only when MikroORM is present, including request contexts, relation loading,
  and optional Kysely integration. Preserve adopted project architecture and
  tooling; introduce additional layers or CQRS only for actual responsibilities.
---

# NestJS Conventions

Write idiomatic TypeScript in cohesive Nest feature modules. Make business rules,
execution order, and ownership clear. Keep the implementation proportionate to
the feature while preserving its validation, authorization, and consistency rules.

## Working rules

- Read the task, applicable `AGENTS.md`/`CLAUDE.md`, nearby implementations, package manifests, lockfile, and formatter configuration first. Honor project-specific conventions over these defaults.
- Check installed versions before choosing framework, validation, GraphQL, ORM, or migration APIs. Preserve the HTTP adapter, module format, decorator/compiler settings, package manager, and test runner.
- Apply structural changes within the requested scope. Keep small fixes small; do not reorganize unrelated features or replace working dependencies.
- Keep only useful roles, contracts, and files. Do not generate a complete layer stack, interface/implementation pairs, or state-identical mappings to satisfy a naming table.
- Follow the repository's language policy for source and tests. Respond in the user's language and report actual verification results.

## Read the relevant references

Read [Code style](references/code-style.md) for every TypeScript implementation or review. Load other references according to the changed behavior.

| Concern | Reference |
|---|---|
| Feature boundaries, domain rules, service collaboration, optional CQRS | [Architecture](references/architecture.md) |
| File locations, module imports/exports, DI tokens, provider lifetimes | [Module structure](references/module-structure.md) |
| Vertical readability, naming, types, asynchronous control flow | [Code style](references/code-style.md) |
| REST, runtime validation, DTOs, partial updates, errors | [API contracts](references/api-contracts.md) |
| GraphQL schema reuse, resolvers, authorization, batching, pagination | [GraphQL](references/graphql.md) |
| MikroORM registration, context, Unit of Work, queries, optional Kysely | [MikroORM](references/mikro-orm.md), when used |
| GraphQL results backed by MikroORM entities or relations | [GraphQL and MikroORM](references/graphql-mikro-orm.md), when both are used |
| Atomic changes, concurrency, retries, external effects | [Transactions](references/transactions.md) |
| Test locations, discovery, fixtures, isolation, verification boundaries | [Testing](references/testing.md) |
| SQL, constraints, indexes, schema migrations | [SQL and migrations](references/sql-migrations.md) |

## Implementation defaults

1. **Start with the feature module.** Keep a small feature's controller/resolver and service together, with DTOs and entities where needed. Add responsibility-based subdirectories when they improve navigation. Preserve an adopted layered layout.
2. **Keep entry points thin.** Handle transport concerns at controllers/resolvers, and delegate business execution to an application provider. Keep repositories, queries, and managed-entity changes out of API handlers.
3. **Use direct, meaningful dependencies.** A service can use its own ORM repository or EntityManager. Add a custom repository for cohesive persistence behavior and a named UseCase for a substantial workflow. Collaborate across features through published application contracts.
4. **Model real business rules.** Put invariants and state transitions in domain entities, value objects, or policies. Use plain functions for suitable pure rules. Keep domain behavior independent of transport and I/O; reuse a useful ORM entity without duplicating its state.
5. **Separate reads by need.** Keep reads free of business mutations. Use QueryServices, query repositories, or command/query handlers when their contracts or complexity justify them, or when the project already adopts them. Do not add buses or empty read/write pairs by default.
6. **Use contracts with a purpose.** Prefer `CreateAccountDto`, `UpdateAccountDto`, GraphQL `Input`, and `AccountRepository` names. A feature service may reuse a suitable DTO; introduce a separate application input for a real boundary. Use classes for required runtime metadata and interfaces/type aliases for compile-time contracts.
7. **Make code read from top to bottom.** Group related statements, separate semantic phases with one blank line, keep exceptional paths shallow, and expand complex expressions into meaningful steps. Follow the formatter without sacrificing visible business flow.
8. **Make runtime behavior explicit.** Validate external input, preserve omission versus null, control public-field exposure, isolate request state, and establish the transaction needed by each operation. Use the installed ORM's facilities directly unless a genuine abstraction is needed.

## Completion checks

- Confirm that imports, runtime tokens, configuration, generated schemas, and migration/test discovery match the installed toolchain.
- Review the diff for unnecessary layers, duplicate DTOs, dense expressions, excessive helper hopping, hidden writes, missing awaits, mutable singleton request state, and exposed internal fields.
- Verify the changed contract at the smallest meaningful boundary. Use real persistence, transport, or GraphQL execution when claiming those paths work.
- Run relevant existing checks, report their outcomes, and identify unverified behavior. Do not infer runtime correctness from mocked execution or document checks.
