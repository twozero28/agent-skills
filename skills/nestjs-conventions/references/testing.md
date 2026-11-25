# Testing

## Test locations and filenames

Preserve an established project layout and naming convention. Otherwise use the following defaults relative to the application/package root, including within a monorepo. Create only directories and files that have a real use.

| Kind | Default location and filename | Example |
|---|---|---|
| Unit test | Beside the target source file, `*.spec.ts` | `src/account/application/account.service.spec.ts` |
| Integration test | `test/integration/<feature>/**/*.integration-spec.ts` | `test/integration/account/account.repo.integration-spec.ts` |
| E2E test | `test/e2e/<feature>/**/*.e2e-spec.ts` | `test/e2e/account/account.e2e-spec.ts` |
| Shared fixture | `test/fixtures/**/*.fixture.ts` | `test/fixtures/account.fixture.ts` |
| Shared seeder | `test/seeders/**/*.seeder.ts` | `test/seeders/account.seeder.ts` |
| Test infrastructure and lifecycle support | `test/support/` | `test/support/test-app.ts`, `test/support/test-database.ts` |

- Place a domain unit test beside its model, such as `src/account/domain/account.entity.spec.ts`. Keep the source basename and replace `.ts` with `.spec.ts`.
- Group integration and E2E tests by the owning feature or workflow. Add deeper directories only when navigation benefits; do not reproduce every production layer in test support.
- Classify tests by the boundary they execute. A Service test using real persistence is an integration test; an E2E test exercises the intended public API/application path.
- Keep single-test helpers local. Share fixtures for constructing values, seeders for installing DB test data, and support for application setup, containers, and cleanup only when reuse is useful.

## Test discovery and build scope

- For this default layout, configure independently runnable suites for `src/**/*.spec.ts`, `test/integration/**/*.integration-spec.ts`, and `test/e2e/**/*.e2e-spec.ts`. Adapt these application-relative patterns to the runner's actual root and configuration; do not assume hyphenated suffixes match its default discovery.
- With Jest, inspect `testMatch` or `testRegex` together with `roots` and `rootDir`. With Vitest, inspect `include`, `exclude`, and project/root configuration. Preserve the adopted runner and setup hooks.
- Keep suite selection disjoint. Update the relevant package scripts and CI jobs when adding or moving tests, so required suites are neither omitted nor run twice accidentally.
- Ensure test TypeScript configuration includes tests and their support files with the required aliases and decorator metadata. Exclude colocated specs and `test/` support from production compilation and packaging; production code must not import test helpers.
- Verify discovered files with the installed runner's listing or targeted execution facility. Confirm each intended suite includes its tests and excludes fixture, seeder, and support modules; an empty successful run is not verification.

See [Jest test discovery](https://jestjs.io/docs/configuration#testmatch-arraystring) and [Vitest include patterns](https://vitest.dev/config/include).

## Choose the verification boundary

Select the contract, the execution path that must remain real, and the smallest setup that exercises it. Preserve the project's runner, compiler, decorator metadata, test naming, and existing support utilities.

| Claim | Required path |
|---|---|
| Domain calculation or transition | Actual domain behavior; no Nest container needed |
| Service or UseCase decisions | Actual target logic; collaborators may be substituted |
| Dependency injection and module visibility | Real module registration and token resolution |
| SQL, constraints, mapping, locking | Actual persistence implementation and relevant database behavior |
| Workflow atomicity | Real participating Services, repositories, and transaction integration |
| GraphQL contract and field loading | Executed schema and intended resolver/application path |
| HTTP validation, authentication, serialization | Configured application request pipeline |
| External adapter protocol | Real adapter against a controlled protocol endpoint or approved provider environment |

A direct method call does not establish guard, pipe, interceptor, filter, or schema behavior. A UseCase test with mocked Services establishes coordination, not database atomicity. Keep the target and the claimed integration path real.

## Application setup

- Reuse the application's bootstrap configuration when testing global validation, prefixes, error mapping, or serialization. Importing `AppModule` does not execute setup statements in `main.ts`.
- Initialize the test application with the production HTTP adapter and the required `app.init()`/adapter readiness steps. Close the application and its owned resources afterward.
- Use provider overrides for collaborators outside the verification boundary. Do not disable the authorization, repository, or transaction implementation under test.
- Use the existing test module and setup helpers. Add a shared setup function only when it removes real duplication; do not rebuild a test framework for each feature.
- Keep ordinary tests independent of live paid services and production credentials. Use existing explicitly enabled provider tests when real-provider compatibility is required.

See [Nest testing](https://docs.nestjs.com/fundamentals/testing) for application creation and overrides.

## Scenarios and fixtures

- Use behavior-describing test names and visible Arrange–Act–Assert stages. Keep scenario-defining input and expected results close to the assertion.
- Cover the changed contract's meaningful success, rejection, and boundary cases. Prefer observable results and state over implementation-shaped call assertions; use call assertions when ordering or delivery is the contract.
- Keep fixtures and seeders in test sources. Start with a local helper; share it when callers need the same meaning, defaults, and reason to change.
- Do not add production setters, constructors, factories, visibility, or relaxed invariants solely to make fixtures or mocks convenient.
- Control ownership, tenant, permissions, status, ordering, time, uniqueness, and relevant boundary values. Reproduce randomized failures rather than rerunning until they disappear.
- Use raw HTTP/GraphQL payloads when malformed input is the test subject. Valid request builders need not express invalid syntax or unsupported fields.
- For partial updates, exercise omission, a supplied value, and explicit null through validation and mapping. Verify that defaults do not turn omissions into updates.

## Isolation and asynchronous work

- Make each test runnable alone and repeatedly. Allocate separate resource namespaces or serialize tests when cleanup could affect another test.
- Await asynchronous assertions, DB operations, and scheduled work. Drain or stop workers before cleanup; do not use arbitrary sleeps as synchronization.
- Restore mocks, timers, and mutable state. Use fake clocks deliberately; do not assume fake timers preserve database-driver or ORM batching behavior.
- Keep shared cleanup in support lifecycle hooks and limit it to test-owned resources. The application and a test client may use different DB connections; a test's outer rollback does not automatically cover HTTP requests or worker jobs.
- Initialize and release request/ORM contexts through the same entry points used in production. A global test context must not hide missing context creation in real jobs or requests.

## Persistence and GraphQL checks

Choose only checks relevant to the change:

- Observe commit/rollback results through a fresh context or connection when persistence durability is the claim. An in-memory entity or completed flush is insufficient.
- Call a DB-changing Service independently, then in a composed workflow. Force a later step to fail and verify that all participating writes roll back.
- For ORM plus SQL-builder operations, prove shared transaction participation and read-after-write behavior with real queries.
- Exercise the actual database engine for engine-specific SQL, uniqueness, locks, isolation, and migrations.
- Inspect generated GraphQL schema and execute the selected relationship fields. Include missing/forbidden objects and tenant separation when relevant.
- Measure real queries to verify batching or pagination; assert a justified bound for representative parent counts instead of inferring performance from loader mocks.
- Verify ordered batch results, argument-sensitive keys, mutation invalidation, and isolation across requests. Test per-parent pagination separately from parent pagination.

Use [transactions](transactions.md), [GraphQL](graphql.md), and [GraphQL/MikroORM](graphql-mikro-orm.md) for the corresponding contracts. Tests may seed or inspect repositories directly; this does not change production layer boundaries.

## Execution and reporting

Run the project's targeted checks first, then required CI gates or broader checks needed for a concrete remaining risk. Preserve test discovery and build settings. Report the command, outcome, and any unverified behavior; distinguish skipped tests from passed tests. Do not install a new runner or impose a coverage percentage solely for this convention.
