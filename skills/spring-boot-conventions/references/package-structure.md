# Package Structure and Placement

## Contents

- [Standard categories](#standard-categories)
- [Placement guide](#placement-guide)
- [Persistence and infrastructure](#persistence-and-infrastructure)
- [Configuration ownership](#configuration-ownership)
- [Events and technical entry points](#events-and-technical-entry-points)
- [Models, DTOs, and interfaces](#models-dtos-and-interfaces)
- [Shared code and new categories](#shared-code-and-new-categories)
- [Resources and module exposure](#resources-and-module-exposure)
- [Test sources](#test-sources)

## Standard categories

Choose the owning feature first, then the necessary categories: `api`, `application`, `domain`, `persistence`, `infrastructure`. These are classifications, not five mandatory folders.

A minimal DB-backed feature may be:

```text
carefacility/
├── api/
│   └── CareFacilityController.java
├── application/
│   └── CareFacilityService.java
└── persistence/
    └── CareFacilityRepo.java
```

A larger feature may add:

```text
carefacility/
├── api/
│   ├── CareFacilityController.java
│   └── dto/
├── application/
│   ├── CareFacilityService.java
│   ├── CareFacilityQueryService.java
│   ├── usecase/
│   ├── command/
│   ├── query/
│   └── event/
├── domain/
├── persistence/
└── infrastructure/
    ├── client/
    ├── config/
    ├── messaging/
    └── scheduler/
```

The larger tree illustrates placement, not scaffolding to generate. Omit empty folders, speculative classes, and unused contracts. Omitting an unnecessary package does not remove responsibility boundaries: a feature without DB access needs no `persistence`, but one with DB access must not put its Repo in `api`.

## Placement guide

Locations are relative to the owning feature.

| Responsibility | Default location | Boundary |
|---|---|---|
| REST Controller / GraphQL DataFetcher | `api` | Delegate business composition |
| Transport requests/responses | `api`, optionally `api/dto` | Not application dependencies |
| Service / QueryService | `application`, optionally `application/service` | Cohesive business responsibility |
| UseCase | `application` or `application/usecase` | No direct persistence access |
| Business commands/results | `application`, optionally `command` or `dto` | Keep equivalent roles consistent |
| Read inputs/results | `application`, optionally `query` or `dto` | Technology-independent call contracts |
| States, value objects, business policies | `domain` | No duplicate transfer-only model |
| Domain creation/update inputs | `domain`, optionally `domain/dto` | Draft/Patch may be Service inputs |
| Domain facts | `domain/event` | No external SDK or broker format |
| Application completion facts/handlers | `application/event` | Handlers delegate business execution |
| Repos, queries, persistence mapping | `persistence` | No external HTTP clients |
| Persistence-specific DTOs | `persistence`, optionally `persistence/dto` | Distinct from application read contracts |
| DB configuration/converters | `persistence/config`, `persistence/converter` | Close to persistence ownership |
| Provider client and provider DTOs | `infrastructure`, optionally `client/<provider>` | Keep provider shapes inside the adapter |
| HTTP client beans and technical properties | `infrastructure/config` | Separate from business policy |
| Broker protocols and consumers | `infrastructure/messaging` | Delegate application behavior |
| Scheduled/batch entry points | `infrastructure/scheduler` | Own timing and invocation |

Use subpackages when they improve navigation. Do not mandate `dto/service/usecase` or `command/query` combinations across all features, mix equivalent roles arbitrarily, or expose internal types merely because files moved.

## Persistence and infrastructure

DB storage, reads, mapping, and directly related technology belong to `persistence`. Other technical integration belongs to `infrastructure`.

Do not mix `infrastructure/repository`, `infrastructure/database`, and `persistence` in new code. If an existing project consistently uses `infrastructure/persistence`, preserve that arrangement during a small change; migrate only in an explicitly scoped task.

Classify Redis/cache code by responsibility, not product name: a storage implementation may belong in `persistence`; messaging or distributed coordination belongs in `infrastructure`. Explicit project conventions take precedence.

## Configuration ownership

Keep feature-specific client configuration in that feature's `infrastructure/config`. Reuse existing global locations for genuinely shared serialization, authentication, or DB setup.

Distinguish property binding from policy: loading configured matching weights is technical configuration; applying them to evaluate suitability is domain/application behavior. Never hardcode secrets in examples or configuration classes; use project secret-management mechanisms.

Custom properties default to a **service-name namespace, then feature**:

```yaml
catalog:
  storage:
    upload-timeout: 30s
  notification:
    retry-limit: 3
```

Names and values are illustrative, not new operational defaults. Bind related settings with `@ConfigurationProperties` when appropriate. Keep framework/library keys such as `spring.*` and `server.*` in their original namespaces.

Preserve existing environment injection and deployment contracts. A property-namespace convention is not permission to rename deployed environment variables.

## Events and technical entry points

```text
Business fact                  -> domain/event or application/event
Business reaction              -> application/event
Broker receive/send protocol   -> infrastructure/messaging
Scheduled invocation           -> infrastructure/scheduler
```

Give each fact one owner; do not duplicate the same event in domain and application packages. Follow an existing published-event contract location where applicable.

Listeners and schedulers invoke UseCase/Service rather than repositories directly. Keep acknowledgement, deserialization, and scheduling separate from business decisions. Transaction and delivery behavior belongs in [Transactions](transactions.md#post-commit-processing).

## Models, DTOs, and interfaces

Use the narrowest meaningful owner. Avoid global collections of every feature's `dto`, `model`, or `interfaces`.

Default to separate files for request/response and business input/result DTOs, including records. A simple private-to-one-class type may be nested. Split it out when reuse, validation, conversion, or output controls grow; no fixed field/file threshold is required.

A few DTOs can sit directly in `api`, `application`, `domain`, or the relevant provider/persistence package. Add a local `dto` package when navigation improves. Keep an established `command/query` layout. Do not combine every layer's DTOs in one feature-level `dto` package.

```text
account/
├── api/
│   ├── AccountController.java
│   ├── RegisterAccountRequest.java
│   └── AccountResponse.java
└── application/
    ├── RegisterAccountUseCase.java
    └── RegisterAccountCmd.java
```

The Cmd in this example represents a distinct application input. When a Service accepts a domain Draft/Patch directly, keep that type in `domain` and omit a copying Cmd.

A record can be a domain value object; syntax does not make it a DTO. Preserve components, validation, serialization, and sensitive-data controls when moving files. Change visibility only as required by the intended contract, not wholesale.

Separate contracts when protocol, security/exposure, validation, or reasons to change differ. Do not duplicate identical fields at every layer. Reuse must preserve dependency direction: domain types cannot depend on API/application inputs. See [Naming](naming.md#input-reuse).

Place interfaces near the responsibility that needs them, not in a global interface folder. Application-owned external capabilities can have implementations in `infrastructure/client`. Repository contract placement follows [Architecture](architecture.md#calls-versus-type-dependencies).

## Shared code and new categories

Share code only when multiple features actually use the same meaning and contract, it does not depend on one feature's internals, and moving it does not conceal a dependency problem. Two callers do not erase business ownership. Distinguish infrastructure capabilities such as clocks or serialization from business features such as approval or subscriptions.

Before adding a sibling category:

```text
Identify feature owner
-> try one of the five categories
-> try a meaningful subpackage
-> document a genuinely independent responsibility only if those do not fit
```

A new client or configuration class does not require a sixth category. `support`, `misc`, or `common` is not a default escape hatch.

## Resources and module exposure

Keep migrations, MyBatis XML, GraphQL SDL, and generated code in the project's existing build/resource layout. Do not move tool resources to imitate Java package paths or edit generated output manually.

For module verification such as Spring Modulith, inspect actual declared interfaces and boundary tests. A Java `public` type is not necessarily a published module contract. Apply `@NamedInterface` and `package-info.java` according to intended exposure, not merely because a subpackage exists. Do not introduce Modulith or expose all internal types to satisfy a layout.

See [module references](source-notes.md#events-and-modules) and [SQL and migrations](sql-migrations.md) as needed.

## Test sources

Mirror production packages under `src/test/java` by default. Keep shared fixtures, seeders, and infrastructure lifecycle code in existing test-support locations or declared test source sets. Do not reproduce all five production categories in test support. [Testing](testing.md) owns the detailed rules.
