# Technical References

## Contents

- [Conventions versus framework behavior](#conventions-versus-framework-behavior)
- [Persistence and concurrency](#persistence-and-concurrency)
- [GraphQL, DGS, and JPA](#graphql-dgs-and-jpa)
- [Spring transactions](#spring-transactions)
- [Events and modules](#events-and-modules)
- [Testing](#testing)
- [Configuration and Liquibase](#configuration-and-liquibase)
- [Agent Skill format and tool integration](#agent-skill-format-and-tool-integration)

## Conventions versus framework behavior

Feature/package structure, UseCase/Service boundaries, DTO suffixes/reuse, Java/SQL layout, configuration namespaces, migration-file immutability, and test/fixture conventions are **design choices for this skill**. They are not mandatory Spring/GraphQL rules or the only valid industry approach.

The official sources below support technical behavior and further verification. Versionless and `current` pages can change; consult documentation matching the project's installed versions. A documentation link is not an instruction to upgrade or add a dependency.

## Persistence and concurrency

| Official reference | Relevant scope |
|---|---|
| [Microsoft: CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) | Separate read/write concerns, including a shared store; distinguish optional messaging/event sourcing |
| [Spring Data: Defining repository interfaces](https://docs.spring.io/spring-data/jpa/reference/repositories/definition.html) | Narrow contracts and repository interface composition |
| [Spring Data: Custom repository implementations](https://docs.spring.io/spring-data/jpa/reference/repositories/custom-implementations.html) | Fragments and implementation discovery/postfix conventions |
| [PostgreSQL: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html) | Isolation, competing updates, and DB-specific consistency guarantees |

`Service/QueryService/Repo/QueryRepo` names and call paths are this skill's conventions. PostgreSQL is a concrete concurrency reference, not a mandatory database or evidence that other engines behave identically.

Draft/Patch ownership and reuse are input-contract choices. Their names do not prescribe an HTTP method, serialization format, or validation library.

## GraphQL, DGS, and JPA

| Official reference | Relevant scope |
|---|---|
| [GraphQL: Thinking in graphs](https://graphql.org/learn/thinking-in-graphs/) | Business-domain types and relationships rather than mechanical persistence exposure |
| [GraphQL: Queries](https://graphql.org/learn/queries/) | Field selections, nested relationships, and fragments |
| [GraphQL: Pagination](https://graphql.org/learn/pagination/) | Pagination wrappers and reuse of object types as nodes |
| [GraphQL: Generating type-safe clients](https://graphql.org/blog/2024-09-19-codegen/) | Operation-specific client types versus schema object types |
| [DGS: Data fetching](https://netflix.github.io/dgs/datafetching/) | Parent/child field resolution and Java result objects |
| [DGS: Data loaders](https://netflix.github.io/dgs/data-loaders/) | Batching, async execution, and security-context considerations |
| [GraphQL Java: Data fetching](https://www.graphql-java.com/documentation/data-fetching/) | Property-based fetching and source objects |
| [Java DataLoader](https://github.com/graphql-java/java-dataloader) | Batch APIs, key/result mapping, cache behavior, and loader lifecycle |
| [Hibernate: LazyInitializationException](https://docs.jboss.org/hibernate/orm/current/javadocs/org/hibernate/LazyInitializationException.html) | Uninitialized state accessed without an appropriate open session |

Reuse business-entity SDL types without requiring one-to-one DB exposure. Projections, DTOs, different exposure contracts, client operation types, and ID-based field resolvers remain valid where appropriate. Check API/runtime compatibility against the project.

## Spring transactions

| Official reference | Relevant scope |
|---|---|
| [Spring: Transaction propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html) | REQUIRED participation, rollback-only, REQUIRES_NEW, resource implications |
| [Spring: Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html) | Declarative behavior, proxy invocation, configurable rollback rules |
| [Spring: @Transactional API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html) | Thread boundaries, read-only hints, propagation/isolation/rollback options |
| [Spring: Proxying mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html) | Proxied calls versus self-invocation |
| [Spring Data JPA: Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html) | Repository defaults and outer application boundaries |

Declaring transactions on Services and on atomic composite UseCases is this skill's boundary policy. Spring does not prescribe those class names or packages.

## Events and modules

| Official reference | Relevant scope |
|---|---|
| [Spring: Transaction-bound events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html) | Event phases and transaction association |
| [Spring: @TransactionalEventListener API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html) | AFTER_COMMIT, fallback execution, remaining transaction resources |
| [Spring: TransactionSynchronization API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html) | Post-commit resource behavior and fresh transactions for further writes |
| [Spring Modulith: Application events](https://docs.spring.io/spring-modulith/reference/events.html) | ApplicationModuleListener and configured publication registry behavior |
| [Spring Modulith: Fundamentals](https://docs.spring.io/spring-modulith/reference/fundamentals.html) | Published module contracts versus internal Java types |

Outbox, retries, and compensation depend on business reliability requirements. This skill mandates neither Modulith nor a broker and does not promise exactly-once delivery. Publication persistence/retry behavior depends on the version and actual configuration.

## Testing

| Official reference | Relevant scope |
|---|---|
| [Spring Boot: Testing applications](https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html) | Full/slice tests and real-port transaction boundaries |
| [Spring: TestContext transactions](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html) | Test-managed transactions, ORM flush, and observation boundaries |
| [Spring: MockMvc overview](https://docs.spring.io/spring-framework/reference/testing/mockmvc/overview.html) | MVC testing without a running HTTP server |
| [Spring Security: MockMvc setup](https://docs.spring.io/spring-security/reference/servlet/test/mockmvc/setup.html) | Including the actual security filter chain in web tests |
| [JUnit User Guide](https://docs.junit.org/current/user-guide/) | Display names, test declarations, and parameterized-test facilities |

Test filenames, display-name adoption/language, mock boundaries, fixture ownership, and production API protection are skill conventions. Dedicated project annotations, enabled infrastructure, and fixture tools remain project decisions.

## Configuration and Liquibase

| Official reference | Relevant scope |
|---|---|
| [Spring Boot: Externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) | Configuration sources and structured property binding |
| [Liquibase: SQL changelog example](https://docs.liquibase.com/community/user-guide-5-0-2/sql-changelog-example) | Formatted SQL header, author/id, and changelog inclusion |
| [Liquibase: What is a changeset?](https://docs.liquibase.com/secure/user-guide-5-1/what-is-a-changeset) | Changeset identity and the distinction between IDs and execution order |

Service-name namespaces, aligned SQL columns, object prefixes, file-number formats, and the policy against appending to applied files are conventions. They are not mandatory DB syntax or claims that Liquibase lacks alternative supported workflows.

## Agent Skill format and tool integration

| Official reference | Relevant scope |
|---|---|
| [Agent Skills specification](https://agentskills.io/specification) | SKILL.md metadata, naming, and supporting files |
| [OpenAI: Build skills](https://learn.chatgpt.com/docs/build-skills) | Skill layout, discovery, and optional agent metadata |
| [OpenAI: Customization](https://learn.chatgpt.com/docs/customization/overview) | Persistent project instructions versus task-specific skills |
| [Claude Code: Skills](https://code.claude.com/docs/en/skills) | Local/project installation and invocation |
