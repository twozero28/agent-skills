---
name: review-code-maintainability
description: >-
  Review code structure, readability, cohesion, responsibility boundaries, dependency direction, testability, and change cost. Use when the user invokes $review-code-maintainability or asks for a maintainability-focused review involving 코드 구조, 유지보수성, 가독성, 책임 분리, 파일 분리, 과도한 책임, or practical refactoring without over-abstraction or excessive fragmentation. Do not use for pure correctness, security, performance, or formatting reviews unless the issue demonstrates a structural maintenance cost.
---

# Review Code Maintainability

Review code as something people must understand, verify, and change safely. Focus on structural and readability costs that exist in the current code, then recommend only the smallest refactoring whose benefit exceeds its added indirection and migration cost.

## Operating Mode

1. Determine the review scope in this order:
   - the exact scope requested by the user;
   - files or symbols explicitly referenced in the request;
   - the relevant staged or unstaged diff;
   - the narrowest feature boundary directly relevant to the request or affected by that diff.
   Inspect as much repository-level structure, architectural context, adjacent callers, and test coverage as needed to understand the requested scope. Keep findings within the requested scope unless the user explicitly asks for a repository-wide review. For a repository-wide review, map the major modules, layers, entry points, and dependency directions before evaluating individual files.
2. Read the applicable repository instructions and inspect the relevant code, call sites, tests, configuration, and dependency boundaries before drawing conclusions.
3. Treat the review as read-only by default. Do not checkout branches, rewrite files, or otherwise mutate the working tree solely to perform a review. Prefer read-only diff and history inspection. Edit code only when the user explicitly asks for fixes or implementation.
4. Keep the review centered on maintainability. Mention a functional bug or style issue only when it demonstrates a structural cause or concrete maintenance risk.
5. Write in Korean by default unless the user requests another language.

## Review Workflow

1. Map the reviewed area before judging individual files:
   - identify entry points, collaborators, data flow, and public boundaries;
   - identify the business rules and orchestration performed in the area;
   - note where tests exercise the behavior and how much setup they require.
2. Identify concrete reading or change tasks a maintainer would perform. Judge whether the current structure helps or obstructs those tasks.
3. Evaluate the review axes below. Verify each suspected issue against actual call sites or tests instead of relying on file size or naming alone.
4. When relevant callers, tests, or repository context are unavailable, distinguish verified findings from plausible but unverified risks. Limit conclusions to the inspected scope and state what evidence would confirm or invalidate the concern. Label unverified risks as `확인 필요` and keep them separate from actionable findings. Do not assign them a priority or disposition. If the available evidence already justifies a change, report the issue as an actionable finding and place only the remaining uncertainty under `확인 필요`.
5. Report one finding per root cause. Combine secondary effects such as readability, test setup, and change blast radius instead of duplicating the issue under multiple axes.
6. Filter every proposal through the refactoring-value rules. Prefer keeping the current structure when separation would add more concepts and navigation than it removes.
7. Report only actionable, evidence-backed findings. Do not manufacture findings to make the review appear complete.

## Review Axes

### 1. Responsibility and Cohesion

- Group responsibilities by independent reasons to change, not by function count or line count.
- Look for business rules, orchestration, persistence, external integration, and presentation concerns changing for different reasons inside one unit.
- Distinguish a large but cohesive file from a file that coordinates unrelated policies or workflows.
- Check whether repeated code represents a repeated business decision that should have one owner, rather than merely similar syntax.
- Check whether a proposed extraction creates a stable, meaningful boundary instead of relocating code.

### 2. Dependency Direction and Change Cost

- Check whether dependencies follow the repository's established architecture and whether lower-level details leak into higher-level policy.
- Look for hidden coupling, circular knowledge, broad imports, and changes that force unrelated callers or tests to move together.
- Check whether important decisions are scattered across modules or duplicated at multiple call sites.
- Evaluate whether a small behavior change requires broad setup, unrelated mocks, or edits across distant files.
- Prefer boundaries that reduce change blast radius while preserving existing public APIs when practical.

### 3. Testability and Behavioral Safety

- Check whether important domain rules can be exercised without unrelated I/O, framework bootstrapping, or broad fixture setup.
- Prefer tests through stable public behavior. Do not require every collaborator to be mockable or hidden behind an interface.
- Distinguish justified integration testing from broad setup caused by mixed responsibilities or unclear boundaries.
- Recommend characterization tests before structural refactoring when existing behavior is insufficiently specified.
- Keep behavior changes separate from structural refactoring unless the user explicitly requests both.

### 4. Readability and Cognitive Load

Treat readability as a maintenance property, not a personal style preference.

- Check whether names communicate domain intent and responsibility rather than only implementation mechanics.
- Check whether the main path reads naturally from top to bottom and whether nesting, branching, boolean flags, or implicit state force mental simulation.
- Keep abstraction levels consistent within a function or module.
- Check whether data creation, transformation, mutation, and ownership remain traceable.
- Keep small, important business rules visible at the call site when extracting them would create a black box.
- Check whether helpers reduce the number of concepts a reader must hold, rather than only reducing line count.
- Prefer comments that explain constraints, decisions, and tradeoffs. Do not request comments that merely restate the code.
- Account for navigation cost in both directions: too much code in one place and closely related code fragmented across too many files.
- Preserve necessary performance-oriented complexity. When it is justified, recommend isolating it behind a clear boundary and protecting it with intent-revealing names, rationale, and tests.

### 5. Refactoring Value

- Tie every proposal to a current cost: slow comprehension, risky edits, excessive test setup, duplicated decisions, or an unnecessarily broad change surface.
- Propose the smallest coherent change that meaningfully reduces that cost.
- Introduce a new boundary only when the boundary itself owns a stable responsibility.
- State what should remain together so the proposal does not encourage further fragmentation.
- Consider migration effort, new indirection, public API churn, and test changes before recommending immediate work.
- Recommend leaving the code as-is when the current cohesion is sound or the proposed abstraction would cost more than it saves.

## Refactoring Guardrails

Do not present the following as findings without additional concrete evidence:

- a file is long or contains many functions;
- a function could technically be extracted;
- one implementation could be placed behind an interface, factory, or strategy;
- code could be moved into a generic `utils`, `helpers`, or `common` module;
- every function, component, or type could have its own file;
- an abstraction might help a hypothetical future requirement;
- formatting, import ordering, or another issue already enforced by tooling;
- a move would satisfy a pattern name without reducing cognitive load or change cost;
- generated files, vendored code, lockfiles, snapshots, or historical migrations are imperfect, unless the user explicitly includes them or they expose a problem in the source that owns them.

Respect framework-imposed structure and repository conventions. Do not recommend a theoretically cleaner pattern when it would make the code less conventional for its ecosystem.

Avoid both under- and over-separation. Consolidation can be the better refactoring when related behavior is scattered across thin wrappers or pass-through helpers.

## Finding Threshold

Report a finding only when the inspected code provides evidence of at least one material cost:

- unrelated reasons to change are coupled;
- dependency direction or ownership is unclear;
- a reader must reconstruct important behavior across excessive indirection;
- control or data flow requires disproportionate mental simulation;
- tests require broad unrelated setup because responsibilities are mixed;
- a local change has an unnecessarily broad impact surface;
- the same domain decision has multiple competing owners.

Do not infer repository-wide ownership, duplication, or impact from a single snippet. Do not assign a numerical maintainability score. Explain the concrete cost and tradeoff instead.

## Response Format

Lead with findings, ordered by expected maintenance benefit and change risk. By default, report only the most important material findings, usually up to five. Provide a broader inventory only when the user requests an exhaustive review. Do not add lower-value findings to reach a target count.

Use `높음`, `중간`, or `낮음` priority only when it helps distinguish action order.

For each finding, include:

- the relevant file or files and the most precise available locator for each: symbol, line range, or diff hunk;
- the concrete maintenance cost;
- the smallest useful change.

Include migration tradeoffs, public API impact, and what should remain together only when they materially affect the recommendation.

After actionable findings, add a separate `확인 필요` section for material unverified risks. Omit the section when empty.

Finish with a short disposition. Omit empty categories:

- **지금 바꿀 부분**: changes whose current maintenance benefit clearly exceeds migration cost;
- **다음 변경 때 처리할 부분**: real costs best addressed when the surrounding code changes;
- **현 구조를 유지할 부분**: cohesive code or abstractions best left alone.

If there are no actionable structural findings, say so explicitly and identify any unreviewed scope or residual uncertainty. Briefly state the reviewed scope and evidence inspected.
