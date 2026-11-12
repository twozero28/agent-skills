# Implementation Workflow

## Contents

- [Inspect the relevant project context](#inspect-the-relevant-project-context)
- [Make a minimal design note](#make-a-minimal-design-note)
- [Choose only necessary structure](#choose-only-necessary-structure)
- [Implement within the agreed scope](#implement-within-the-agreed-scope)
- [Verify the changed behavior](#verify-the-changed-behavior)
- [Review and report](#review-and-report)
- [Decisions left to the project](#decisions-left-to-the-project)

## Inspect the relevant project context

Before editing, inspect the task's relevant scope:

- Applicable root/path `AGENTS.md` and `CLAUDE.md`, build files, Java/Spring versions, and libraries.
- Actual entry points and maintained comparable implementations in or near the feature.
- Persistence, transaction managers, error mapping, formatters, and existing checks.
- For tests: path-specific rules, composed annotations, lifecycle/optional infrastructure, fixtures, and execution requirements.
- For configuration/DB changes: property namespaces, schema/naming conventions, migration layout, and applied history.

Do not assume one similar file establishes the whole project's pattern, or explore the entire repository indefinitely for a small fix. Confirm real paths, classes, settings, and supported versions. Resolve material instruction conflicts without copying defects or expanding the task into an unsolicited migration.

## Make a minimal design note

Use this as a thinking aid; do not print or save a lengthy design document for a small change. Omit irrelevant items.

```text
Business goal and owning feature:
Read/write intent and domain/read model:
Entry point and Service/UseCase flow:
Domain invariant/policy owner:
Existing capability to reuse:
Input contract: reuse Draft/Patch or justify a distinct Cmd/Query:
New responsibilities and files, with reasons:
Atomic DB scope:
External effects and recovery:
Success/failure behavior to verify:
Test type, real path, substitutes, infrastructure, and critical data:
```

Resolve unknowns by inspecting existing code. Do not invent externally visible contracts or consistency policies without support from the task/project. Routine naming and placement do not require repeated confirmation.

## Choose only necessary structure

| Question | Decision |
|---|---|
| One cohesive business responsibility? | API -> Service |
| Independent capabilities need coordination? | Add a meaningful UseCase |
| Controller owns cross-service branching/results? | Extract the workflow; keep transport concerns outside it |
| Existing Service already provides the capability? | Reuse or extend its meaningful contract |
| Core rule, transition, or invariant? | Put it in domain behavior/policy even if simple |
| Data-serving query? | QueryService -> QueryRepo with a suitable result |
| Load/check/lock needed for a command? | Use the command-side Repo |
| Service and domain input meaning match? | Reuse Draft/Patch; omit a copying Cmd |
| Real implementation/module contract boundary? | Introduce an interface only where useful |
| New package? | Start with the five categories and existing subpackages |
| Same GraphQL entity in several views? | Reuse its SDL type; vary selections/fragments |

The table is not a scaffolding checklist. Do not add empty roles or overload an existing Service with meaningless flags just to claim reuse. Detailed decisions belong to [Architecture](architecture.md), [Naming](naming.md), and [GraphQL and JPA](graphql-jpa.md).

## Implement within the agreed scope

Define meaningful call contracts, connect Service data preparation/domain behavior/persistence, and keep a composite workflow understandable from its UseCase. Use QueryService for read results. Follow the project's TDD and implementation sequence.

Keep DTOs near their owner, omit copying stages, and prevent provider/ORM/SQL mechanics from leaking into business contracts. Preserve established serialization, errors, authorization, and module boundaries. Use [Package structure](package-structure.md) when placement changes.

Do not change library versions, build layout, public response shapes, authentication, or global error policies incidentally. For uncertain technical behavior, inspect actual code and official documentation for the installed version; do not assume a particular documentation tool or plugin exists.

Apply relevant [Code style](code-style.md) rules without unrelated reformatting or a new formatter. For configuration and schema work, follow existing operational contracts and [SQL and migrations](sql-migrations.md).

Do not mechanically introduce domain copies for rule-free data movement, paired read/write classes, interfaces, mappers, events, separate databases, or buses. Each added element must solve a current responsibility or boundary problem.

## Verify the changed behavior

Use [Testing](testing.md) to define what each test proves and preserve that path. Reuse project support; do not alter production APIs for fixtures or mock away the contract being verified.

| Change | Relevant verification |
|---|---|
| Pure domain rule | Success, rejection, boundaries |
| Optional partial update | Omitted/value/null semantics across input conversion and domain behavior |
| UseCase composition | Collaborator contracts, failure propagation, business outcome |
| Persistence/mapping | Actual queries, storage, constraints, DB-specific behavior |
| Concurrent transition | Invariants under competing requests; conflict/conditional-update behavior |
| Transaction boundary | Real proxied calls, commit/rollback, post-commit behavior |
| External adapter | Protocol mapping, timeout/failure, retry/duplicate contract |
| API | Validation, relevant authentication/authorization, response contract |
| GraphQL/JPA relationships | Type reuse, loading, actual batching, paging, access, request isolation |

Do not substitute call counts for meaningful outcomes or add tests tightly coupled to irrelevant internals. Do not impose a new framework, mock product, coverage number, or source set.

Use the project's build wrapper and required command order. Without an explicit sequence, choose relevant compilation, formatting, focused tests, and necessary integration checks. Do not mandate `clean`, all tests, or live-provider tests for every change.

Inspect actual command outcomes. A blocked mandatory check is not complete because another test passed. Distinguish passed, failed, skipped, and unexecuted work; never report a command as run when it was not.

## Review and report

Inspect the diff and relevant [review criteria](review-checklist.md). Check whether a reader can find responsibilities and follow execution without traversing arbitrary helpers, and whether simplification bypassed a required boundary.

Prioritize findings:

1. Correctness, data integrity, transaction, and authorization risks.
2. Responsibility, contract, and test defects.
3. Naming, readability, and formatting.

Do not lead with taste when there is an operational risk, or propose architecture replacement where a local fix suffices.

Report briefly:

```text
Change: business behavior added or changed
Flow: entry point -> UseCase/Service -> persistence or external adapter
Verification: commands actually run and their outcomes
Limits: skipped/unexecuted checks and remaining risks
```

Explain any new package, interface, special transaction, or boundary exception. Do not repeat an entire document or codebase when the user can inspect the actual diff. Use the user's language.

## Decisions left to the project

This skill does not select exception frameworks/error codes, HTTP/GraphQL error envelopes, dependency versions, persistence tools, response formats, final import tooling, numerical length limits, test infrastructure/coverage, messaging products, outbox storage, or retry operations.

Resolve a missing decision only when the task requires it. Do not build a large common framework merely to fill unspecified policy fields.
