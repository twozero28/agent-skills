---
name: nestjs-conventions
description: >-
  Apply when implementing, changing, refactoring, testing, or reviewing TypeScript
  NestJS backends: feature modules, dependency injection, domain rules,
  UseCase/Service/Repository boundaries, code-level CQRS, REST/GraphQL contracts,
  validation, transactions, naming, code style, migrations, and tests. Apply the
  MikroORM and GraphQL/MikroORM references only when those technologies are used,
  including request contexts, relation loading, and optional Kysely integration.
  Preserve project conventions; do not impose NestJS on other stacks or introduce
  an ORM, GraphQL, or an architecture migration without a task requirement.
---

# NestJS Conventions

Organize by feature, put core business rules in domain models or policies, separate read and write responsibilities in code, and use UseCases for meaningful orchestration. Keep file locations, contracts, and execution flow predictable. Create abstractions only for responsibilities that exist.

## Scope

- Read the task, applicable `AGENTS.md`/`CLAUDE.md`, package manifests, lockfile, and adopted formatter first. Project-specific rules take precedence over these defaults.
- Check the installed NestJS, TypeScript, validation, GraphQL driver, ORM, and migration versions before choosing APIs. Preserve the HTTP adapter, module format, decorator/compiler settings, package manager, and test runner.
- Apply structural rules to new or substantially changed production code. Keep small fixes small; do not reorganize the application or replace dependencies incidentally.
- Reuse maintained project patterns without reproducing defects. Explain material exceptions briefly in the delivery, not through unrelated code changes.
- Keep fixtures and test support in test sources; do not generate production layers for them. Respond in the user's language and follow the repository's language policy for code and tests.

## Load relevant references

| Task | Read |
|---|---|
| Feature responsibilities, domain modeling, CQRS, orchestration | [Architecture](references/architecture.md) |
| File placement, module imports/exports, providers and injection tokens | [Module structure](references/module-structure.md) |
| Naming, TypeScript, formatting, asynchronous code | [Code style](references/code-style.md) |
| REST, DTO validation, partial updates, errors, authentication | [API contracts](references/api-contracts.md) |
| GraphQL schema, resolvers, batching, pagination, authorization | [GraphQL](references/graphql.md) |
| MikroORM entities, context, Unit of Work, loading, optional Kysely | [MikroORM](references/mikro-orm.md), only when MikroORM is used |
| GraphQL entity results, ORM wrappers and relationship resolution | [GraphQL and MikroORM](references/graphql-mikro-orm.md), when both are used |
| DB writes, composed operations, concurrency, external effects | [Transactions](references/transactions.md) |
| Tests, fixtures, isolation, execution boundaries | [Testing](references/testing.md) |
| SQL, constraints, indexes, migration files | [SQL and migrations](references/sql-migrations.md) |

Load only references relevant to the change. GraphQL, MikroORM, Kysely, and a CQRS bus are optional technologies, not prerequisites.

## Core boundaries

| Responsibility | Call path |
|---|---|
| One cohesive change | Controller/Resolver → Service → Repo |
| Read contract | Controller/Resolver → QueryService → QueryRepo |
| Composite workflow | Controller/Resolver → UseCase → Services/QueryServices |
| Core business rule | Service → Domain model or policy |
| Background entry point | Consumer/Scheduler → UseCase or Service |

- Use feature modules with `api`, `application`, `domain`, `persistence`, and `infrastructure` categories as needed. Keep persistence separate from other infrastructure unless the project already has a consistent alternative.
- Keep controllers/resolvers focused on transport and conversion. Do not access repositories, mutate managed entities, or coordinate independent business services there.
- Let a UseCase own ordering, conditions, combined results, and workflow atomicity. It must not inject or call repositories, ORM clients, EntityManagers, SQL executors, or persistence mappers, including for reads. An execution-only transaction boundary is a valid collaborator.
- Give each Service one cohesive business responsibility. Load state, establish authorization/context, invoke domain behavior, and persist through its own collaborators. Compose independent business services in a UseCase; policies and limited technical capabilities are valid Service collaborators.
- Put invariants, calculations, and state transitions in domain models or policies. Keep I/O and application calls out of them. Reuse a useful ORM domain entity instead of making a state-identical copy.
- Keep query contracts free of business mutations. Command-side state loading, existence checks, and locking belong in the command Repo; not every SELECT needs a QueryService.
- Collaborate across features through published application contracts. Nest module exports are deliberate contracts; do not re-register another module's providers to bypass them.

## Contracts and persistence

- Own DTOs near their layer. Use `Request/Response`, `Cmd/Query`, `View/Result`, `Cond`, and `Draft/Patch` by purpose; do not create every conversion stage.
- Reuse domain-owned `Draft/Patch` inputs when their meaning and validation match. Application contracts must not depend on API `Request/Input` classes; domain contracts must not depend on application commands.
- Validate external values at runtime. TypeScript types alone do not validate input or supply every DI/schema token. Preserve omitted, supplied, and explicit-null states in partial updates.
- Reuse one GraphQL server type for the same business concept and exposure contract across list, detail, and search. Separate types need a different meaning, exposure policy, or structural purpose.
- Define transaction ownership for independently callable changes and atomic workflows. Confirm that composed repositories use the same transaction; a request context or DI scope alone does not provide atomicity.
- Keep raw persistence facilities out of application method inputs and outputs. Use the project's transaction mechanism or a narrow runner that exposes execution without exposing an ORM client.
- Make relationship loading, public-field exposure, batching, authorization, and cache lifetime explicit. ORM metadata and serialization are not substitutes for an API contract.

## Implementation workflow

1. Inspect a nearby implementation and identify the affected feature, entry points, contracts, and installed technologies.
2. Classify the work as a read, cohesive change, or composite workflow. Identify invariants, access rules, transaction boundaries, and external effects before adding layers.
3. Load the relevant references and implement the smallest complete change. Keep configuration, imports, runtime tokens, generated schema, and migration discovery consistent.
4. Select verification by the behavior at risk. Exercise real DB, transaction, transport, or GraphQL paths when making claims about those paths.
5. Review the diff for unnecessary layers, direct UseCase persistence access, mutable singleton request state, missing awaits, leaked fields, and broken module boundaries.
6. Run relevant existing checks and report actual results, distinguishing passes, failures, and checks not run. Do not claim runtime behavior from document checks or mocked execution alone.
