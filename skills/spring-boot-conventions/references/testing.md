# Testing: Make the Verification Boundary Explicit

## Contents

- [Scope and precedence](#scope-and-precedence)
- [Choose what the test proves](#choose-what-the-test-proves)
- [Reuse project test configuration](#reuse-project-test-configuration)
- [Mocks, stubs, and spies](#mocks-stubs-and-spies)
- [Structure and readability](#structure-and-readability)
- [Isolation and lifecycle](#isolation-and-lifecycle)
- [Test data and fixtures](#test-data-and-fixtures)
- [Requests and real external systems](#requests-and-real-external-systems)
- [Transactions and failure paths](#transactions-and-failure-paths)
- [Project-specific decisions](#project-specific-decisions)

## Scope and precedence

Apply these rules to `src/test/**` and explicitly declared test-support source sets such as `integrationTest` or `testFixtures`. Do not create source sets solely for this skill.

Read root/path-specific instructions and existing build/test support first. Project rules own TDD, command order, required completion checks, annotation names/options, infrastructure, and cleanup implementation. This reference supplies general verification and isolation principles, not a replacement test runner or execution sequence.

Do not silently relax a project policy when it conflicts with a convenient test arrangement; resolve its scope explicitly.

## Choose what the test proves

Select **contract -> real execution path -> necessary context/infrastructure -> replaceable collaborators**. Choose the smallest setup that actually satisfies the verification goal, not the smallest setup at any cost.

| Goal | Typical setup | Path that must remain real |
|---|---|---|
| Calculation, transition, invariant | Unit test without Spring | Target domain behavior |
| Service/UseCase branching and collaboration | Unit test with suitable substitutes | Target Service/UseCase logic |
| SQL, mapping, constraints, repository behavior | DB slice or repository integration test | Data access and required DB behavior |
| Combined Services, transactions, events | Application integration test | Relevant Spring beans, proxies, repositories, resources |
| GraphQL schema execution, resolvers, selected result shape | Project GraphQL execution test | Intended schema/resolver/query path |
| MVC mapping, validation, message conversion, configured filters/security | MockMvc or equivalent project web test | Actual required MVC/filter/security configuration |
| Real port, container, transport/network behavior | Real-server request test | Required deployed HTTP path |
| External adapter protocol, serialization, errors | Adapter integration/contract test | Real adapter and relevant client path |

A UseCase unit test can replace collaborator Services to verify order, arguments, and failures, but it does not prove their real business work or DB atomicity. Add integration verification for risks that require it.

In-process GraphQL execution does not automatically exercise HTTP filters/authentication. MockMvc can exercise configured security and serialization without a real port, but only when those components actually participate. Real-server testing is for the behaviors that require it, not a universal requirement whenever the word “security” appears.

## Reuse project test configuration

Use existing composed test annotations and support configurations appropriate to the goal. Inspect their real declarations and options; do not invent available infrastructure.

- When the project standardizes a composed setup, do not rebuild its containers and policies independently with raw base annotations.
- Without such a setup, `@SpringBootTest`, `@DataJpaTest`, and similar base annotations are valid. Do not introduce a new annotation framework only for this skill.
- Narrow slices using the project's supported extension mechanisms, such as `@Import`. Do not mock every missing bean merely to make the context start.
- Distinguish baseline from optional infrastructure; activate only what the intended path needs.
- Preserve deliberate fail-fast behavior for disabled infrastructure. Select the correct option/setup or narrow the declared goal rather than hiding misconfiguration with a mock.
- Reuse the adopted test engine, mocking, and assertion tools. Avoid duplicate libraries for one test.

Do not remove necessary DB, security, filters, or adapter paths merely because a test is slow.

## Mocks, stubs, and spies

Choose substitutes by the verification boundary, not by a class suffix.

| Test | Permitted substitution | Invalid claim/result |
|---|---|---|
| Service unit test | Repository and external capabilities | Mocking the target Service and asserting its programmed result |
| UseCase unit test | Collaborating Services | Claiming real DB atomicity from call verification |
| Repository integration | Unrelated external systems | Replacing the target repository/DB path |
| Business integration | Out-of-scope side effects, explicit failure injection | Mocking the internal behavior being verified |
| HTTP adapter | Local protocol stub server/emulator | Mocking the HTTP client, then claiming wire serialization works |
| Real-provider integration | Explicitly permitted auxiliary isolation | Replacing the provider, then claiming provider compatibility |

An adapter test need not use paid cloud credentials. State what a local stub/emulator proves; use existing opt-in provider tests when actual provider compatibility matters.

Make failure injection explicit. A Spy may invoke real side effects; inspect its behavior. Isolating an unrelated side effect must not erase the promised test path or conceal a broken normal path.

Verify calls when calls themselves are the contract. Do not substitute invocation counts for required state, results, or failure invariants.

## Structure and readability

Default to production-package mirroring under `src/test/java` with `*Test.java`. Follow an established `*IT` pattern or source-set discovery rule when present; renaming must not remove tests from execution.

For JUnit Jupiter, add a behavior-describing `@DisplayName` to each test method, not only the class. Follow the project's language policy; otherwise use concise Korean display names. Do not migrate another test engine to enforce this annotation.

```java
@Test
@DisplayName("Approving a pending facility changes its status to approved")
void approvePendingFacility() {
    var facility = pendingFacility();

    facility.approve();

    assertEquals(FacilityStatus.APPROVED, facility.status());
}
```

Use actual project domain construction, fixtures, and assertions; the example is not a reason to add a production factory.

Make Given-When-Then or Arrange-Act-Assert visible through names and blank lines. Use stage comments only where helpful or required. One test covers one coherent scenario; multiple assertions for that scenario are valid.

Select important success, rejection, boundary, and post-failure contracts. Keep critical inputs and expected outcomes visible instead of hiding them in giant shared setup. Exercise public behavior rather than adding reflection or changing visibility to call private methods. Preserve actual duplicate-handling, ordering, and time contracts.

Do not force CRUD method order or public visibility onto tests. Group related scenarios, put private helpers below them, and use nested/parameterized tests when they clarify actual scenario structure or repetition.

## Isolation and lifecycle

Each test must run alone and repeatedly without earlier test data, ordering, or mutable global state. An intentional lifecycle test that verifies cleanup can be an explicit ordering exception; do not generalize it to business tests.

Inspect isolation for resources actually used: DB, caches, search indexes, buckets, files, and background work. Do not start every possible service or prescribe one universal cleanup strategy.

- Shared cleanup belongs to test-support lifecycle code, not copied truncate/bucket/index-reset logic in each test.
- A test may own and clean local temporary resources, without duplicating shared cleanup or deleting another test's resources.
- Verify test-only databases, schemas, keys, buckets, and indexes. Never clean production or shared development resources.
- Parallel tests need safe namespaces, ports, and data ownership. Whole-resource cleanup is not automatically compatible with parallel execution.
- Await/cancel background work in a deliberate order so it cannot mutate the next test's resources. Do not rely on arbitrary long sleeps.

A test method's rollback does not clean all resources. Real-server requests, separate threads, independent transactions, and external stores have their own boundaries. In real-port Spring Boot tests, the server transaction is separate from the test thread's rollback. See [testing sources](source-notes.md#testing).

## Test data and fixtures

Prepare only the scenario's required data. Write simple values explicitly; use an existing object generator for genuinely complex incidental structure. Do not make FixtureMonkey or another generator mandatory.

Control all values that determine constraints or assertions: required columns, status, ownership, permission, time, boundaries, ordering, and duplicates. Unique values must avoid unintended collisions; a duplicate-rejection test deliberately uses duplicates.

Use established fixed-clock/time injection facilities. If random data causes a failure, retain reproducible input or a seed and resolve the cause; do not rerun until it passes by chance.

### Placement and reuse

```text
Used in one test class
-> private helper in that class

Repeated in multiple classes with the same meaning, shape, and reason to change
-> existing shared fixture/seeder support

Existing shared test source set
-> reuse it
```

Two callers alone do not justify shared ownership. As a supplementary default, a fixture builds objects/values and a seeder persists/installs test data; preserve established project terminology.

Avoid universal builders, boolean-heavy seeders, and giant TestUtils. Shared support can supply safe incidental defaults, but scenario-defining values remain explicit in the test or arguments.

### Production API protection

Keep data-generation support in test sources. **Do not add production constructors, factories, setters, or change signatures/visibility/invariants solely for fixture or mocking convenience.** Legitimate business changes or separately scoped design refactoring are different; state their purpose and impact.

Fixtures and postcondition checks may access repositories or test DB tools directly. This is not the production UseCase rule. Direct seeding must not bypass the registration, authorization, or validation path that this particular test claims to verify.

## Requests and real external systems

Prefer existing GraphQL codegen, typed request builders, and test clients for valid requests. Do not duplicate normal requests as raw strings without a reason.

**Use a raw request when malformed syntax, unsupported fields, or invalid payloads are the test subject and the typed builder cannot express them.** Make that intent visible. Ask about an exception only when the project explicitly prohibits the required raw form or defines an approval process; the mere absence of an exception policy is not a reason to stop.

Keep real KMS/cloud/paid API tests behind their existing dedicated configuration and explicit activation conditions. Do not require live credentials or trigger external effects during ordinary tests. Report skipped tests and their verification gap; a mocked test does not prove real-provider integration.

## Transactions and failure paths

Select relevant checks from [Transactions](transactions.md#verification): standalone proxied Service commit/failure, composite rollback, propagation and rollback policy, concurrency/constraints, conditional-update conflicts, success-only effects, and retry/duplicate handling.

Ensure the test's transaction does not replace an application transaction that is missing in production. For ORM tests, use appropriate flush, reload, or a separate verification transaction to observe DB constraints and durable state. **Flush is not commit.** State what boundary the assertion observes.

Mocks alone do not prove proxies, SQL, commit, or rollback. Use the actual engine/features when DB-specific behavior is the subject.

For optional field updates, test omission, supplied values, and explicit null through deserialization, conversion, and domain behavior. Do not impose an optional-field contract on a Patch whose selected properties are all intentionally required.

For GraphQL/JPA changes, verify relevant loading, batching, parent paging, argument semantics, authorization, and cache isolation described in [GraphQL and JPA](graphql-jpa.md#verification).

## Project-specific decisions

Do not hardcode another project's annotation names, enabled services, fixture implementation, cleanup strategy, or execution commands into this generic skill.

| Project owns | This skill requires |
|---|---|
| Composed annotations and options | Inspect and select the actual configuration |
| DB/cache/search/object-storage stack | Preserve the required real path and safe isolation |
| Fixture libraries and support types | Reuse them without redundant dependencies |
| Live integration activation | Explicit opt-in and secret/resource safety |
| TDD, Gradle/Maven commands, CI gates | Follow scoped instructions and report actual outcomes |

When working in CareLink or another project with dedicated test guidance, preserve those exact contracts instead of regenerating generic replacements. File naming, display-name adoption, mock boundaries, fixture ownership, and production API protection are conventions here, not universal framework mandates.
