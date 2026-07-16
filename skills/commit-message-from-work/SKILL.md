---
name: commit-message-from-work
description: Use when the user asks for a copy-paste-ready Conventional Commit message from the current work, conversation context, staged changes, git diff, or a specified file/module/range. Write the subject summary and body in the user's requested or inferred language while keeping the Conventional Commit type and scope in English or stable domain identifiers. Triggers include requests for a commit message, title and body, or a message for a specific scope or range.
---

# Commit Message From Work

Create one copy-paste-ready Conventional Commit message in the user's selected language. Keep the Conventional Commit type and scope in English or stable domain identifiers.

This skill is for turning the current work into a commit message. If the user does not name a target, summarize the work done so far. If the user names a file, module, feature, staged area, commit range, branch range, or other scope, write the message only for that target.

## Language Selection

Choose the language for the subject summary and body in this order:

1. Use the language explicitly requested by the user.
2. Otherwise, use the agent's configured response language or the language already established for responses in the current conversation.
3. If no preference is established, use the language the user is currently using.
4. If the language is still unclear, use English.

Apply the selected language consistently to the subject summary and all body prose. Preserve API names, entity names, file names, table names, and other technical identifiers in their original form.

## Default Output

Return a single fenced `text` block:

```text
feat(notification): <summary in the selected language>

- <concrete change in the selected language>
- <another concrete change in the selected language>
```

Do not add explanation before or after the block unless the user asks for alternatives, asks for reasoning, or the scope is ambiguous enough to require a brief note.

## Scope Selection

Use the user's explicit scope when present:

- a file, directory, module, feature, docs-only slice, or test-only slice
- staged changes, unstaged changes, or other work intended for a commit
- a named commit, branch, range, PR/MR, issue, or ticket
- comparisons such as `origin/main...HEAD` or `HEAD~3..HEAD`

If no explicit scope is present:

1. Prefer the recent conversation/work in this thread.
2. If a repo is available, inspect non-sensitive git state to summarize actual current changes.
3. If both conversation and git state are insufficient, ask one concise question instead of inventing details.

## Inspection Rules

When the current repo state is needed, use read-only commands first:

- `git status --short`
- `git diff --stat`
- `git diff --cached --stat`
- `git diff --name-only`
- targeted `git diff -- <path>` when details are needed
- targeted range diffs such as `git diff origin/main...HEAD --stat` when the user gives a range

Do not read `.env*`, private keys, credentials, DB dumps, certificates, token files, or other sensitive files unless the user explicitly approves.

Do not stage, commit, reset, clean, push, or modify files as part of this skill unless the user separately asks for that action.

## Subject Rules

Use:

`type(scope): summary in the selected language`

Common types:

- `feat`: new behavior, API, UI, capability, or workflow
- `fix`: bug fix or incorrect behavior correction
- `refactor`: structure change without intended behavior change
- `docs`: documentation-only change
- `test`: test-only change
- `build`: build, dependency, toolchain, deployment packaging
- `chore`: maintenance that does not fit the above

Choose the most specific honest type. Keep the type in lowercase English. Keep the scope short and domain-based, using English or an established project identifier such as `notification`, `carefacility`, `hospital`, `dashboard`, `auth`, `search`, `build`, `docs`, or `test`.

Write the summary in the selected language. Keep it short, idiomatic, and review-friendly rather than translating a fixed phrase mechanically.

## Body Rules

- Use only `- ` bullet lines.
- Use 4 to 8 bullets by default.
- Each bullet should describe one concrete change.
- Write body prose in the selected language.
- Preserve important domain terms, entity names, API names, file names, or table names from the work.
- Mention tests only when they are present in the user input, conversation, or inspected diff.
- Do not invent tests, verification, APIs, files, tables, or behavior.
- Avoid vague bullets equivalent to `feature improvements`, `code cleanup`, or `add tests` without specifics.
- Avoid marketing language.
- Do not switch to English-only prose unless English is the selected language. Technical identifiers may remain in their original language.

## Split Commit Heuristic

If the scoped work clearly belongs to separate commits, briefly say that splitting would be cleaner and show the most useful split. But if the user asks for one commit message, produce exactly one message.

Split-worthy boundaries include:

- docs-only changes plus runtime code
- dependency/toolchain upgrade plus product behavior change
- test infrastructure rewrite plus product behavior change
- unrelated frontend and backend changes
- multiple independent features

## Final Check

Before answering, verify:

- The output is copy-paste-ready.
- There is exactly one subject line unless the user asked for alternatives.
- The subject summary and body use the selected language consistently.
- The Conventional Commit type and scope remain English or stable domain identifiers.
- The body bullets match the selected scope.
- No unsupported claim was added.
- Sensitive file contents were not exposed.
