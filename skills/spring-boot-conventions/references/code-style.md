# Java Style: Code That Reads Like Clear Prose

## Contents

- [Scope and defaults](#scope-and-defaults)
- [Class member order](#class-member-order)
- [Private helpers](#private-helpers)
- [Method flow](#method-flow)
- [Multiline declarations and calls](#multiline-declarations-and-calls)
- [Braces and conditions](#braces-and-conditions)
- [Chains and lambdas](#chains-and-lambdas)
- [Annotations](#annotations)
- [Types and records](#types-and-records)
- [Imports](#imports)
- [Comments](#comments)
- [Exceptions and logs](#exceptions-and-logs)
- [Change scope and tests](#change-scope-and-tests)

## Scope and defaults

Follow explicit project rules, the adopted formatter/static checks, and maintained local conventions first. These are defaults, not universal Java standards. Preserve behavior and keep unrelated reformatting out of functional changes.

Use four spaces per indentation level and one extra level for continuation lines unless the project says otherwise. This does not require literal tabs. Prefer predictable file layout and execution flow over the fewest characters.

Do not run an unadopted formatter/preset and treat its output as an existing project rule. Check configuration before formatting. Do not introduce a formatter replacement or absolute line/method-length limits solely for this skill.

## Class member order

Where CRUD grouping is natural:

```text
Constants and fields
Constructors / object-creation static factories

Public business methods:
  Read -> Create -> Update -> Delete

Private implementation helpers:
  Grouped by the public methods' call flow
```

Group related operations and overloads within each category, not alphabetically. Classify by business effect, not name prefix or SQL: approval is Update; a business deletion implemented as a status change is still Delete. Inspect ambiguous operations such as cancellation or withdrawal rather than renaming them to fit a category.

Do not add category comments, empty groups, or missing CRUD operations. Place new methods appropriately without rearranging unrelated members during a small fix.

Do not force CRUD order onto UseCases centered on `execute`, policies, calculations, event handlers, or tests. Keep lifecycle methods and overrides such as `equals/hashCode` together according to project practice. Constructors/static factories are class construction, not business Create operations.

Never change access modifiers or field/initializer execution order to improve visual sorting. If a large public surface is hard to navigate, inspect cohesion rather than splitting solely by method count.

## Private helpers

Default to public business methods first and private helpers below them, not alternating public/private methods. Inside the private region, group helpers by the public-method flow; callers precede callees. Keep a shared helper once, normally near its first business caller's helper group.

```text
public getById
public search
public register
public approve
public delete

private normalizeSearch
private validateRegistration
private validateApproval
private hasRequiredDocuments
```

“Keep related methods close” refers to grouping within the lower private region by default. An explicitly adopted caller-adjacent project style takes precedence. Place protected/package-private members by their real contracts; do not change visibility to sort them.

Extract meaningful rules, complex calculations, or repeated logic. Do not scatter three straightforward steps across three one-use helpers, hide simple calls behind utilities/base classes, or split only to meet a line limit. Keep the main business flow visible at its entry point. Vague names such as `process1` indicate an unclear responsibility, not successful extraction.

## Method flow

Prefer a readable sequence of input checks, data preparation, rule evaluation, business action, persistence/follow-up, and result. Load data before checks that require it; do not invent absent stages.

```java
@Transactional
public ApprovalResult approve(UUID facilityId, UUID actorId) {
    var facility = repository.getById(facilityId);
    approvalPolicy.validate(facility, actorId);

    facility.approve();
    repository.save(facility);

    return new ApprovalResult(facility.id(), facility.status());
}
```

This fragment assumes an explicit save contract and existing project types. Do not add redundant JPA saves merely to match it. Keep related statements together and separate logical stages and adjacent methods with one blank line. Do not insert blank lines after every statement or compress unrelated stages into one block.

## Multiline declarations and calls

Keep short, clear declarations/calls on one line. For a long declaration, call, or constructor invocation:

- Break immediately after `(`; put one parameter/argument per line.
- Indent arguments one level; do not align them using method-name-length padding.
- Put closing `)` at the declaration/call's starting indentation: `) {` or `);`.
- Do not leave the first argument on the opening line or mix several arguments on subsequent lines.

```java
@Transactional
public ApprovalResult approve(
    UUID facilityId,
    UUID actorId,
    FacilityApprovalContext approvalContext
) {
    var facility = repository.getById(facilityId);
    approvalPolicy.validate(facility, actorId, approvalContext);

    facility.approve();
    repository.save(facility);

    return new ApprovalResult(
        facility.id(),
        facility.status(),
        approvalContext.approvedAt()
    );
}
```

This is a separate layout example, not a request to add parameters or overloads. Argument count alone does not require wrapping. Do not add temporary values or DTOs only to shorten a line. Apply the same indentation recursively; extract a meaningful intermediate value when nesting obscures intent, preserving evaluation order, frequency, and effects.

Avoid:

```java
return new ApprovalResult(facility.id(),
                          facility.status(), approvalContext.approvedAt());
```

## Braces and conditions

Always brace `if`, `else`, `for`, enhanced `for`, `while`, and `do` bodies, even for one statement. `else if` is valid. Express intentionally empty loop bodies with braces. Adding braces must preserve the associated `else` and loop extent.

```java
for (var item : items) {
    item.validate();
}
```

Keep short conditions on one line. When wrapping, break after `(`, use one logical unit per line, place continuing `&&`/`||` at the start of the next line, and align closing `)` with the `if`:

```java
if (
    facility.isPending()
    && facility.hasRequiredDocuments()
    && (actor.isAdministrator() || actor.manages(facility.id()))
) {
    facility.approve();
}
```

Use parentheses to make mixed boolean groupings explicit. Follow an adopted formatter where it differs.

Use guard clauses to reduce unnecessary nesting, but preserve the original business contract. Returning early for an already-approved object is correct only when duplicate approval is an accepted success, not when it must be rejected.

Name a condition when it expresses a useful business concept. Do not proliferate one-use booleans merely to shorten code. Preserve short-circuiting, null guards, evaluation order/count, and effects: extracting a call must not execute it before a guard that previously prevented it. Avoid nested ternaries for multiple business branches.

## Chains and lambdas

Use streams for clear transformation, filtering, or aggregation. Do not hide saves, external calls, or multiple mutations in long streams; prefer a loop when clearer.

For multiline chains, put one operation on each continuation line with a leading dot:

```java
return facilities.stream()
    .filter(CareFacility::isApproved)
    .map(CareFacilitySummaryView::from)
    .toList();
```

Do not wrap every short property access. Split long lambdas/deep chains into meaningful stages. Apply multiline argument rules inside builders. Use only syntax/APIs supported by the project's Java version.

## Annotations

### Shared layout

Put class/method annotations immediately above the declaration, one per line, without arbitrary blank lines or padding. Order Javadoc -> annotations -> declaration.

### Classes: logging first, constructor generation last

Order by reading role, not library origin:

```text
Logging
-> Class role
-> Role settings / conditions / behavior
-> Other declaration support, when needed
-> Constructor generation
-> Class declaration
```

```java
@Slf4j
@Service
@Primary
@RequiredArgsConstructor
public class NcpKmsEncryptionServiceHttp implements EncryptionService {
    // Implementation omitted in this layout example.
}
```

For a controller, the same principle gives `@Slf4j`, `@RestController`, `@RequestMapping`, then `@RequiredArgsConstructor`.

Keep logging at the top and constructor-generating annotations directly above the class. Do not group `@Slf4j` and `@RequiredArgsConstructor` at the top merely because both are Lombok. Role/settings include stereotypes, `@Primary`, `@Profile`, mappings, and class-level transaction declarations.

Documentation, getters, builders, and similar support normally precede constructor generation, following comparable project classes. Not every Lombok annotation generates a constructor. Keep genuinely needed constructor annotations together in the final group; do not add any to complete the pattern. Do not impose Service annotations on DTOs, entities, or configuration classes.

Use consistent order within each role group without inventing a total ordering for every annotation. Interpret composed annotations by their actual role.

### Methods: show the invocation contract first

Default order:

```text
@Override
-> API mapping / event / test entry point
-> Authorization
-> Transaction / cache / execution declarations
-> Documentation or auxiliary declarations
```

For example, `@Override` precedes `@Transactional`; `@Test` precedes `@DisplayName`.

This is **source layout**, not AOP precedence or runtime ordering. Do not reorder repeated annotations, array elements, explicit ordering settings, or other potentially meaningful sequences merely for style. Do not add logging, `@Primary`, transactions, or dependencies to copy an annotation example.

Do not force this grouping onto field, parameter, or type-use annotations. Short forms such as `@Valid Request request` may remain inline without changing the annotation's target.

### Long annotation attributes

Keep short annotations on one line. For long attributes, use one per line and align `)` with the annotation start:

```java
@RequestMapping(
    path = "/care-facilities",
    produces = MediaType.APPLICATION_JSON_VALUE
)
```

Preserve attribute values and element order. Adopted formatter/project rules take precedence.

## Types and records

Use `var` when the right-hand side and context make the type/role clear; otherwise improve the name or use an explicit type. Default to constructor injection and final dependency fields. Follow the project's Lombok policy.

Prefer immutable DTOs where supported; use records only when Java, serialization, and contract requirements fit. Do not expose setters indiscriminately or pass API requests into domain behavior. Preserve the project's nullability/validation system and do not upgrade Java/Spring to use a syntax preference.

Record components default to one per line, indented one level. Put closing `)` on its own line at the declaration indentation. Even an empty body uses separate `) {` and `}` lines:

```java
public record ItemView(
    UUID id,
    String name
) {
}
```

Apply this shape even to a one-component record. Put constructors/methods in the body when needed. Do not convert a class to a record, reorder components, or change annotations merely for formatting. Separate-file versus nested placement follows [Package structure](package-structure.md#models-dtos-and-interfaces).

## Imports

Separate `package`, imports, and the class body with blank lines. Remove unused imports and do not add wildcard imports prohibited by the project.

Without configured grouping, use:

```text
java.*

javax.* / jakarta.*

Spring and other external libraries

Project packages

lombok.*

Static imports
```

Let the IDE/formatter sort within groups. Existing formatter rules, including static-import placement, take precedence. This is a local default, not a universal Java standard.

## Comments

Explain non-obvious reasons, constraints, or business background rather than restating code. Follow the project's language convention; otherwise use concise Korean comments.

Use brief stage comments when genuinely helpful, not universal “validate/process/return” templates or CRUD region markers. Add Javadoc for public contracts or surprising behavior where needed, not boilerplate getter documentation.

## Exceptions and logs

Reuse the project's exception hierarchy, error codes, and API error mapping. Do not add a new exception framework merely to apply this skill.

Do not swallow failures or convert them silently into empty/success results. Preserve causes when translating exceptions at meaningful boundaries. Avoid logging the same failure in every layer.

Log diagnostic context and business identifiers, not secrets, tokens, passwords, or unnecessary personal data. Distinguish expected business rejection from system failure rather than assigning both the same severity by habit.

## Change scope and tests

Include only cleanup needed for the task. Add libraries, utilities, and base abstractions only for actual problems existing code cannot reasonably solve. Similar lines do not prove shared meaning or a shared reason to change; local duplication can be clearer than the wrong abstraction.

Do not force CRUD order or public visibility on test methods. Group related scenarios, show preparation -> action -> assertions, and put private test helpers below scenarios. [Testing](testing.md) owns test behavior.

For style-only changes, inspect the diff for altered visibility, initialization order, short-circuiting, evaluation frequency, exceptions, transaction behavior, returned values, annotation targets, and annotation settings. Formatting is not authorization to refactor architecture.
