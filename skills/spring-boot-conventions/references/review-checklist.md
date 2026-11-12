# Review Checklist and Decision Scenarios

Use only criteria relevant to the change. This is a review aid, not a requirement to create missing layers, tools, or tests. Detailed rules remain in the linked references.

## Contents

- [Responsibilities and contracts](#responsibilities-and-contracts)
- [GraphQL and persistence](#graphql-and-persistence)
- [Transactions and effects](#transactions-and-effects)
- [Readability and operational contracts](#readability-and-operational-contracts)
- [Tests and verification](#tests-and-verification)
- [Decision scenarios](#decision-scenarios)
- [Optional automated checks](#optional-automated-checks)

## Responsibilities and contracts

See [Architecture](architecture.md), [Package structure](package-structure.md), and [Naming](naming.md).

- [ ] Feature ownership and each new responsibility are clear; only needed categories/subpackages exist.
- [ ] API delegates business behavior; growing orchestration is extracted by workflow, not by dependency count alone.
- [ ] UseCases do not access Repos, QueryRepos, SQL executors, ORM sessions, or persistence mappers.
- [ ] Services are cohesive business capabilities; independent-service orchestration is not hidden inside one.
- [ ] Domain behavior owns core rules/transitions/invariants; listeners/schedulers do not duplicate them.
- [ ] Cross-feature calls use published contracts, not internals; package movement does not expose every type.
- [ ] Read/write intent is clear, command-side SELECTs remain valid, and read paths preserve authorization without mutating business state.
- [ ] DTOs/interfaces/domain models solve real needs. Equivalent Draft/Patch inputs are reused without API-to-application or application-to-domain dependency inversion.
- [ ] Optional updates preserve omission/value/null; required-property Patches remain valid.
- [ ] DTO ownership, nesting, and local grouping improve navigation without a global dumping ground.
- [ ] UseCases do not mutate returned entities or rely on implicit dirty checking.

## GraphQL and persistence

See [GraphQL and JPA](graphql-jpa.md).

- [ ] List/detail/search reuse the same business SDL type unless meaning or exposure really differs.
- [ ] Client operation types and useful server projections are not mistaken for duplicate SDL types.
- [ ] Entity/relationship reuse preserves the public contract without exposing technical/sensitive fields.
- [ ] Graph traversal is not confused with JPA association requirements, aggregate ownership, or write permission.
- [ ] Query results remain safe across persistence-context boundaries; no accidental lazy getter access is required.
- [ ] DataLoaders call published application bulk contracts, not per-key DB queries or another feature's Repo.
- [ ] Batch keys/results, missing values, collection emptiness, and errors follow actual loader/schema contracts.
- [ ] Parent selection, ordering, and pagination precede child hydration; argument-sensitive relationship pages remain correct.
- [ ] Loader cache scope/keys and actor/tenant propagation preserve isolation and authorization.
- [ ] Async work does not assume inherited thread-bound transactions or sessions.

## Transactions and effects

See [Transactions](transactions.md).

- [ ] Standalone DB-changing Services have required boundaries; atomic composite UseCases have an outer boundary.
- [ ] The intended manager, resources, proxies, and execution threads actually participate.
- [ ] Self-invocation, checked exceptions, rollback-only state, and `REQUIRES_NEW` are understood rather than hidden.
- [ ] Synchronous post-commit writes use an actual new transaction; asynchronous/Modulith boundaries are not duplicated blindly.
- [ ] External effects are not treated as DB-rollback participants; durability, retries, duplicates, and recovery match the need.
- [ ] Domain checks are supported by necessary constraints, version/conditional updates, and affected-row handling under concurrency.

## Readability and operational contracts

See [Code style](code-style.md) and [SQL and migrations](sql-migrations.md).

- [ ] Names express business role and necessary technology; framework discovery names such as fragment `Impl` are preserved.
- [ ] Public CRUD methods/private helpers, multiline arguments/conditions, braces, records, imports, and annotations follow applicable project defaults.
- [ ] Logging is first and constructor generation last where relevant; visual order is not mistaken for execution precedence.
- [ ] Style-only changes preserve visibility, initialization/evaluation order, null guards, exception/transaction behavior, and annotation settings.
- [ ] Helpers, chains, and abstractions improve rather than hide the business flow; comments explain reasons.
- [ ] Custom settings preserve service/feature ownership, framework keys, secret handling, and deployment contracts.
- [ ] New SQL follows adopted layout/naming; applied files and changesets remain unchanged, and subsequent execution order is valid.
- [ ] No unrequested version/tool/formatter/API-contract migration is included.

## Tests and verification

See [Testing](testing.md).

- [ ] Scope-specific test instructions, support configurations, optional infrastructure, and command order were inspected.
- [ ] Each test states a coherent contract and preserves the real path it claims to verify.
- [ ] Unit-test substitution is distinguished from removing an integration path; the target itself is real.
- [ ] GraphQL execution, MockMvc/security, and actual server/network coverage are distinguished.
- [ ] Invalid raw requests are used when necessary without needless confirmation; explicit project prohibitions remain respected.
- [ ] Critical inputs, constraints, time, sorting, and expected outcomes are visible and controlled.
- [ ] Shared cleanup, parallel/background work, and test-only resource ownership are safe.
- [ ] Fixtures stay in test sources; production APIs/invariants were not weakened for convenience.
- [ ] Test transactions do not mask missing production boundaries; flush is not claimed as commit.
- [ ] Live-provider activation is respected; passed, failed, skipped, and unexecuted checks are reported accurately.

## Decision scenarios

| Request or situation | Expected decision | Avoid |
|---|---|---|
| Add facility-name editing | Reuse/extend the owning Service and Repo | Pass-through UseCase or domain copy |
| Update facility details/options in three tables | One Service if one responsibility | Three Services merely because three tables exist |
| Register account and facility atomically | UseCase + shared local transaction | Controller composition or separate commits |
| Controller coordinates branches and results | Cohesive workflow UseCase | Giant dependency-hiding UseCase or HTTP objects moved into it |
| Request/response records obscure a controller | Separate files near `api` | Moving every layer's DTOs into one package |
| Small feature has few DTOs | Keep them near their owner; simple internal types may nest | Mandatory `dto` hierarchy |
| Query maps directly to a DTO | QueryRepo -> QueryService | Repository bypass or business writes in reads |
| Command needs current state/existence/lock | Read through its own Repo | Detour through QueryService for every SELECT |
| Simple but important approval rule | Domain behavior plus needed DB protection | Wait until complexity/duplication to use Domain |
| Service input equals domain creation values | Accept Draft directly | Identical Cmd-to-Draft copy |
| Skip redundant API/application transfer stage | Map to an application/domain-owned contract | Service accepts API Request/Input |
| Create new object with explicit-storage persistence | Construct from values, then save | Load a nonexistent object first |
| REST and business inputs differ in meaning | Separate meaningful contracts | Merge because field counts match |
| Field-by-field optional update | Preserve absent/value/null | Ignore all nulls or clear all omitted values |
| Add a standalone Service call path | Verify Service transaction and rules | Assume a UseCase always supplies the transaction |
| Implement an NCP KMS HTTP capability | Meaningful provider/interface/mechanism name | Generic `EncryptionServiceImpl` |
| Concrete Repo needs no interface | Keep role-named concrete class | Invent interface to justify a technical prefix |
| Spring Data custom fragment implementation | Keep configured discovery postfix | Rename away `Impl` and break discovery |
| Feature-specific client configuration | Feature's infrastructure/config | Move everything to global configuration |
| Create a simple feature | Only necessary packages | Five empty category folders |
| Post-commit notification | Match durability/retry/duplicate requirements | Claim async/durable/exactly-once from timing alone |
| Synchronous post-commit audit write | Actual new transaction through a valid boundary | Assume ordinary REQUIRED necessarily starts fresh |
| Service uses encryption contract | Treat as a limited collaborator | Force UseCase because both names end in Service |
| Small legacy-monolith bug fix | Local change and relevant verification | Whole-project package migration |
| Reorder read/approve/delete methods only | Read -> Update -> Delete, private helpers below | CQRS redesign or empty Create methods |
| Classify approval/cancel/soft delete | Inspect business effect | Classify solely by prefix or SQL UPDATE |
| Wrap long arguments/conditions | Preserve guards, effects, and evaluation order | Horizontal padding or eager extracted effects |
| Reorder class annotations | Logging -> role/settings -> constructor | All Lombok at top or AOP behavior inferred from order |
| Add record without adopted formatter | Use this skill's record layout | Apply a new preset and call it project policy |
| Application package becomes crowded | Add useful local subpackages | Mandatory layout in every feature or broader exposure |
| Improve new DDL readability | Align per-table name/type/attribute columns | Reformat applied history or apply Java no-padding rule |
| Add migration in date-based project | Existing pattern, new file, correct ID/include | Convert history or append to applied file |
| Service unit test | Real target, suitable mocked collaborators | Force real DB into every unit test |
| Composite rollback integration test | Real proxies/Services/DB, explicit observation | Mock core Service or mask missing boundary with test transaction |
| Disabled optional infrastructure fails fast | Select intended setup/option | Hide misconfiguration with a mock |
| Verify HTTP request serialization | Actual adapter/client + local protocol stub | Mock client and claim wire compatibility |
| Fixture needs another object state | Test helper and existing creation path | Test-only production constructor/setter |
| Similar fixtures occur twice | Share only same meaning/shape/change reason | Universal boolean-driven seeder |
| Real-port test changes DB | Cleanup accounting for server transaction | Rely solely on test-thread rollback |
| Apply project-specific test guide | Preserve exact annotations/options/tools | Replace concrete support with generic scaffolding |
| Order list/detail/search select different fields | Same Order SDL type, distinct selections | OrderListItem/OrderDetailResponse without distinct meaning |
| Client codegen produces operation-specific types | Keep them | Remove valid client types under SDL reuse policy |
| Monthly sales statistics are requested | Purposeful aggregate read result | Force statistics into an entity type |
| GraphQL Order.customer references another feature | Resolver/loader -> published bulk QueryService | Direct foreign Repo/entity coupling |
| JPA query returns before relationship resolution | Explicit fetch/materialization or ID-based resolution | Accidental lazy loading outside the session |
| DataLoader receives many IDs | True bulk query and correct result mapping | One query per ID inside a “batch” loop |
| Child condition affects parent page membership | Apply predicate/order in parent selection | Load a parent page, then discard parents in memory |
| Same relationship is queried with different arguments | Argument-aware key/query semantics | Cache by parent ID alone |
| Async loader needs actor and DB access | Explicit context propagation and proper query boundary | Assume caller thread context/transaction follows |
| Invalid field cannot be represented by codegen | Raw negative request with clear intent | Stop for permission without an actual prohibition |
| Configured security/filter behavior is the test target | Appropriate MockMvc/web test | Require a real port for all security tests |
| Formatting-only task includes transaction annotations | Preserve values and behavior | Change propagation while “sorting” annotations |

## Optional automated checks

Use existing ArchUnit, Spring Modulith verification, formatters, and static checks. Do not introduce them merely because this checklist mentions them.

Encode actual ownership and contracts, not naming shortcuts. A blanket ban on all `*Service` references can incorrectly block encryption or other technical capabilities. Test architectural rules against real code before claiming they enforce the intended boundary.
