---
name: spring-boot-conventions
description: >-
  Apply when implementing, changing, refactoring, testing, or reviewing Java/Spring Boot
  code: feature packages, domain rules, UseCase/Service/Repository boundaries, code-level
  CQRS, GraphQL/JPA, transactions, naming, Java/SQL style, configuration, and migrations.
  Preserve project conventions and reuse domain-centered GraphQL types across views.
  Do not impose Spring architecture on other stacks or migrate a codebase without a request.
---

# Spring Boot Conventions

Make code location, responsibility, and execution flow predictable. Organize by feature, keep core rules in the domain, separate reads from writes at the code level, and use UseCases for meaningful orchestration. Apply DDD where it clarifies business concepts and invariants; do not generate layers just to complete a pattern. No persistence library is mandatory.

## Scope and precedence

- Read the task, applicable `AGENTS.md`/`CLAUDE.md`, build configuration, and adopted formatter first. This skill supplements project rules; it does not silently override them.
- Apply boundaries to new or structurally changed production code. Keep small fixes small; do not migrate packages, dependencies, or architecture incidentally.
- Reuse maintained project patterns, not their defects. Explain material conflicts or necessary exceptions briefly.
- **Required** means a boundary within this convention's scope. **Default** applies without a project-specific rule. **Optional** requires a real need.
- Test-support rules apply to test sources and declared test source sets. Do not reproduce production layers in fixtures or seeders.
- This skill's language does not change the user's response language or the repository's language policy for comments, test names, and business text.

## Load only relevant references

Do not load the entire bundle for a small change. Each reference owns its detailed rules; checklists are reminders, not additional requirements.

| Task | Read |
|---|---|
| Feature implementation or structural change | [Workflow](references/implementation-workflow.md), [Architecture](references/architecture.md) |
| GraphQL schema, DGS resolvers, JPA relationships | [GraphQL and JPA](references/graphql-jpa.md) |
| Package, DTO, event, or configuration placement | [Package structure](references/package-structure.md) |
| Class names, interfaces, input reuse, partial updates | [Naming](references/naming.md) |
| Java changes or formatting | Relevant sections of [Code style](references/code-style.md) |
| DB writes, transaction boundaries, external effects | [Transactions](references/transactions.md) |
| Tests, fixtures, isolation | [Testing](references/testing.md) |
| DDL, indexes, migration history | [SQL and migrations](references/sql-migrations.md) |
| Review or final verification | Relevant items in [Review checklist](references/review-checklist.md) |
| Framework behavior or supporting documentation | [Source notes](references/source-notes.md) |

## Core boundaries

```text
Single responsibility: API -> Service -> Repo
Read:                  API -> QueryService -> QueryRepo
Composite workflow:    API -> UseCase -> Services -> their Repos
Core rules:            Service -> Domain model or policy
Technical entry point: Scheduler / Consumer -> UseCase or Service
```

- Use feature-first packages. Standard categories are `api`, `application`, `domain`, `persistence`, and `infrastructure`; create only those needed. Keep DB access in `persistence`, separate from other infrastructure, unless the project consistently uses another arrangement.
- API code handles transport, input conversion, and responses. It does not access repositories, mutate domain state, or orchestrate independent business services.
- A UseCase owns execution order, conditions, combined results, and any workflow-wide transaction. It must not inject or call repositories, ORM sessions, SQL executors, or persistence mappers, including read-only ones. Do not add pass-through UseCases.
- A Service owns one cohesive business responsibility, not one table. It prepares data and authorization context, invokes domain behavior, and persists changes. Orchestrate independent business services in a UseCase; policies and limited technical capabilities are valid collaborators.
- Domain models or policies own core rules, state transitions, calculations, and invariants from the outset. They do not call repositories, HTTP APIs, or application services. Do not duplicate a useful JPA domain entity merely to remove persistence annotations.
- Repositories own persistence and mapping. Command-side loading, existence checks, and locking reads belong there; not every SELECT needs a QueryService.
- QueryService/QueryRepo serve read contracts and enforce access/exposure rules without changing business state. Code-level CQRS does not require another database, event sourcing, or a bus.
- Cross-feature collaboration uses published application contracts, not another feature's repositories or internal entities. Respect stricter module rules where present.

## Contracts and GraphQL

- Place DTOs in separate files near their owning layer by default. Use nested types only for simple internal details; add `dto` subpackages when navigation benefits.
- Use `Request/Response`, `Cmd/Query`, `View/Result`, `Cond`, and `Draft/Patch` by purpose. Do not generate every conversion stage.
- Reuse domain-owned `Draft/Patch` directly as Service input when fields, meaning, and validation match. Service/UseCase inputs must not depend on API `Request/Input`; domain types must not depend on application `Cmd/Query`.
- Preserve omitted, supplied, and explicit-null states for optional partial updates. A Patch need not make every field optional.
- Model GraphQL around business entities and relationships. When JPA entities represent them well, use that model as the default. Reuse the same server type for list, detail, and search; selection sets and fragments express view-specific fields.
- Add GraphQL types for genuinely different concepts or exposure contracts, aggregates of query results, pagination, or operation results. Reuse existing business types inside wrappers. Client operation types and internal projections are not duplicate server types.
- JPA entities may be resolver results when appropriate; a matching DTO is not mandatory. Plan loading, session lifetime, authorization, batching, and parent-level pagination explicitly. Graph traversal does not define aggregate or write boundaries.
- Keep SQL executors, generated persistence records, and ORM sessions out of business contracts. Introduce interfaces for real contracts, module boundaries, or implementation selection, not merely for mocks.

## Transactions and external effects

- Independently callable DB-changing Services declare their required transaction. A UseCase also declares one when composed changes must succeed or fail together.
- Default to `REQUIRED` under the same local manager, participating DB resources, and synchronous proxy-mediated execution. Do not assume the same atomicity across threads, managers, or databases.
- Check self-invocation, rollback rules, rollback-only state, and `REQUIRES_NEW`. Catching an exception does not restore a rollback-only transaction.
- External effects are not undone by DB rollback. Keep DB transactions short and design required recovery, idempotency, and delivery durability.
- A synchronous `AFTER_COMMIT` callback that writes to the DB needs an actual new transaction, such as a separate proxied bean with `REQUIRES_NEW`; default `REQUIRED` alone is not a safe assumption.

## Readability, tests, and delivery

- Default suffixes: `Config`, `Service/Repo`, `QueryService/QueryRepo`. Name implementations by meaningful provider/technology. Preserve framework discovery names such as Spring Data fragment `Impl`.
- Where CRUD ordering fits, group public methods Read -> Create -> Update -> Delete; place private helpers below them. Use four-space indentation, braces, vertical multiline arguments, and multiline record components. Class annotations: logging first, role/settings next, constructor generation last. Follow the adopted formatter without changing behavior.
- Select tests by the contract and real execution path they must prove. Do not mock away that path or change production constructors, setters, visibility, or invariants solely for test convenience.
- Reuse project test infrastructure and execution rules. Control important data and time; isolate resources. Distinguish GraphQL execution, MockMvc/security, and real-server coverage; flush is not commit.
- Group custom properties by service and feature; preserve framework keys and deployment contracts. Follow existing migration conventions. Do not edit or append changesets to already-applied files; add subsequent changes in new files.
- Before delivery, review the diff and run relevant project checks. Report passed, failed, skipped, and unexecuted checks separately. Never claim compilation, integration coverage, or agent evaluation that did not run.
