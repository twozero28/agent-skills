# Naming and Interfaces

## Contents

- [Naming priorities](#naming-priorities)
- [DTO names](#dto-names)
- [Input reuse](#input-reuse)
- [Patch semantics](#patch-semantics)
- [Implementation names](#implementation-names)
- [When to introduce an interface](#when-to-introduce-an-interface)
- [Methods and variables](#methods-and-variables)

## Naming priorities

Name by **business meaning -> responsibility -> necessary implementation identity**. Use consistent patterns for equivalent roles.

| Role | Default example |
|---|---|
| REST entry point | `CareFacilityController` |
| GraphQL entry point | `CareFacilityDataFetcher` |
| Write capability | `CareFacilityService` |
| Read capability | `CareFacilityQueryService` |
| Composite workflow | `RegisterCareFacilityUseCase` |
| Command-side persistence | `CareFacilityRepo` |
| Read persistence | `CareFacilityQueryRepo` |
| Configuration | `SecurityConfig` |
| Completed business fact | `CareFacilityRegistered` |

Use `Config` for handwritten configuration classes and `Repo` for repository classes/interfaces by default. Keep framework names such as `@Configuration`, `@Repository`, and `JpaRepository`. A property-binding `Properties` type has a different role and need not become `Config`.

Use `AccountService/AccountRepo` for writes and `AccountQueryService/AccountQueryRepo` for reads. Do not add `Cmd` or `Command` to service/repository names; `Cmd` names a separate write input. Do not rename an existing `command` package merely to match the abbreviation.

Honor existing project naming. Do not mass-rename types, create empty read/write pairs, or introduce classes to complete the naming table. “Repository” in prose describes a role, not a mandatory class suffix.

## DTO names

Place types near their owning feature/layer; see [Package structure](package-structure.md#models-dtos-and-interfaces).

| Owner | Purpose | Suffix | Example |
|---|---|---|---|
| `api` | REST request | `Request` | `CreateAccountRequest` |
| `api` | Separately needed REST response | `Response` | `AccountResponse` |
| Existing GraphQL/codegen location | Schema input | `Input` | Follow SDL/codegen conventions |
| `application` | Write input | `Cmd` | `ChangePasswordCmd` |
| `application` | Read input | `Query` | `SearchAccountQuery` |
| `application` | Read result | `View` | `AccountSummaryView` |
| `application` | Separately needed write result | `Result` | `CreateAccountResult` |
| `persistence` | Query/search conditions | `Cond` | `AccountSearchCond` |
| `domain` | Domain creation values | `Draft` | `AccountDraft` |
| `domain` | Selected domain-property changes | `Patch` | `AccountProfilePatch` |

Do not abbreviate REST `Request/Response`. GraphQL types retain schema conventions; do not append REST response suffixes. This table does not change an API's public response format or require a DTO per row.

Return an ID directly when that is the entire creation result. Add `Result` when a meaningful structured result is needed. Use `Cond` only for actual condition objects; do not wrap every scalar ID or saved object. Do not rename domain value objects into DTOs to satisfy the table.

## Input reuse

`Cmd` describes a business request; `Draft` describes domain creation values; `Patch` describes selected property changes.

**When fields, meaning, and validation match, a Service may accept a domain-owned Draft/Patch directly.** Do not add a state-identical Cmd simply because the call crosses an application layer.

```text
AccountService.create(AccountDraft)
    -> Account.create(AccountDraft)

AccountService.update(accountId, AccountProfilePatch)
    -> account.update(AccountProfilePatch)
```

Use a Cmd when actor, target, expected version, or other context forms a distinct request, or meaning/validation differs. It may contain a Patch. One extra scalar does not automatically justify a wrapper. Prepare any required creation values before constructing the final domain input.

Do not require every stage of `Request -> Cmd -> Draft`, `Query -> Cond`, or `View -> Response`. Separate actual protocol, security, validation, exposure, and change boundaries, not suffixes. Equal field counts or names do not make different business meanings interchangeable.

Service/UseCase inputs remain application- or domain-owned. They must not accept API `Request` or generated GraphQL `Input` types directly. The API can map a Request/Input to a domain Draft/Patch when no Cmd is needed. Shared contracts flow from API toward application/domain; domain types must not depend on API `Request/Input` or application `Cmd/Query`.

## Patch semantics

A Patch changes **selected properties of an object's overall state**; it does not prescribe an HTTP method or wire format. A name/email Patch can require both fields. Do not make every Patch field optional by definition.

When individual fields are optional, preserve all three states end to end:

| Input state | Meaning |
|---|---|
| Omitted | Keep the current value |
| Supplied value | Replace with that value |
| Explicit `null` | Clear it if allowed; otherwise reject |

Do not collapse omission and explicit null during deserialization or mapping. Reuse the project's presence representation and validation; no wrapper library is mandatory. Test the complete conversion path when these semantics change.

Approval, cancellation, and other core transitions are business actions, not generic `Patch(status)` updates that bypass invariants. Prefer a meaningful command where needed and a domain behavior such as `account.approve()`.

## Implementation names

For an actual interface, default to:

```text
[Provider/technology] + [Interface name] + [Optional implementation mechanism]

EncryptionService <- NcpKmsEncryptionServiceHttp
FileStorage       <- S3FileStorageSdk
TokenStore        <- RedisTokenStore
```

`Http` or `Sdk` may identify a meaningful implementation path even when only one implementation exists. Omit redundant detail; do not enumerate the entire stack. Keep project acronym casing consistent; these examples use `NcpKms`, `Http`, and `Sdk`.

A standalone concrete `CareFacilityRepo` needs no technical prefix and no interface created solely for naming. Conversely, a real storage contract can justify a provider-specific implementation name.

Avoid habitual `XxxServiceImpl`, `DefaultXxx`, `CommonXxx`, `BaseXxx`, `Util`, `Manager`, or `Helper` when they hide rather than describe responsibility. A genuinely defined default, base contract, or manager responsibility can be valid.

**Framework discovery is an explicit exception.** Preserve Spring Data custom repository fragment names such as `OrderSearchFragmentImpl` when the configured implementation postfix is `Impl`. Do not rename a framework extension point just to remove that suffix. Verify the project's actual configuration before changing it; see [repository references](source-notes.md#persistence-and-concurrency).

## When to introduce an interface

Introduce one for a real implementation choice, provider boundary, published module contract, or meaningful dependency direction. One implementation can still justify an interface. A mock test or a speculative future replacement alone does not justify interface/implementation pairs everywhere.

With multiple implementations, make bean selection deliberate using the project's qualifiers, conditions, or explicit configuration. Do not attach `@Primary` routinely to every implementation or rely on accidental naming.

A `Service` suffix does not prove a class is an application business Service. Apply collaboration rules by responsibility; see [Architecture](architecture.md#service-collaboration).

## Methods and variables

Use capability names such as `approve`, `register`, `cancel`, and `search`. `execute` fits a UseCase whose class already names the workflow.

Keep absence contracts consistent: follow the project's distinction between optional `findById`, throwing lookups, and reference-returning framework methods. Similar method names do not guarantee identical behavior.

Distinguish IDs from objects: `accountId`, `facilityId`, `registrationResult`. Avoid vague long-lived `data`, `obj`, and `result1`; a local `result` is fine when its meaning is clear. Boolean names should read naturally, such as `isActive`, `hasPermission`, or `canApprove`; avoid opaque flags and nested negations.
