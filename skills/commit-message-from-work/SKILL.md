---
name: commit-message-from-work
description: Use when the user asks for a Korean copy-paste-ready Conventional Commit message from the current work, conversation context, staged changes, git diff, or a specified file/module/range. Triggers on Korean requests like "커밋메세지 추천", "커밋제목이랑 본문", "지금까지 한 거 커밋 메시지", "복붙가능한 커밋 메시지", and requests for a commit message for a specific part or range.
---

# Commit Message From Work

Create one copy-paste-ready Conventional Commit message in Korean.

This skill is for turning the current work into a commit message. If the user does not name a target, summarize the work done so far. If the user names a file, module, feature, staged area, commit range, branch range, or other scope, write the message only for that target.

## Default Output

Return a single fenced `text` block:

```text
feat(notification): 비회원 알림함과 current owner API 추가

- 회원 알림 조회/구독을 current owner API로 추가
- NotificationClient 등록 및 push endpoint client 연결
- guest_notifications 기반 비회원 조회/구독/읽음 처리 추가
- NotificationClient 동시 등록 upsert 처리
- 알림 query/realtime/repository 테스트 보강
- 알림 저장과 채널 발송 책임 분리 원칙 문서화
```

Do not add explanation before or after the block unless the user asks for alternatives, asks for reasoning, or the scope is ambiguous enough to require a brief note.

## Scope Selection

Use the user's explicit scope when present:

- "이 파일", "이 모듈", "notification 쪽", "docs만", "테스트만"
- "staged", "스테이징된 것", "커밋할 것"
- "main 대비", "origin/main 기준", "HEAD~3..HEAD"
- a named commit, branch, directory, file path, PR/MR, issue, ticket, or feature

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

`type(scope): Korean summary`

Common types:

- `feat`: new behavior, API, UI, capability, or workflow
- `fix`: bug fix or incorrect behavior correction
- `refactor`: structure change without intended behavior change
- `docs`: documentation-only change
- `test`: test-only change
- `build`: build, dependency, toolchain, deployment packaging
- `chore`: maintenance that does not fit the above

Choose the most specific honest type. Keep the scope short and domain-based, such as `notification`, `carefacility`, `hospital`, `dashboard`, `auth`, `search`, `build`, `docs`, or `test`.

Keep the Korean summary short and review-friendly. Prefer endings like `추가`, `수정`, `정리`, `개선`, `보강`, `분리`, `연동`, `마이그레이션`.

## Body Rules

- Use only `- ` bullet lines.
- Use 4 to 8 bullets by default.
- Each bullet should describe one concrete change.
- Preserve important domain terms, entity names, API names, file names, or table names from the work.
- Mention tests only when they are present in the user input, conversation, or inspected diff.
- Do not invent tests, verification, APIs, files, tables, or behavior.
- Avoid vague bullets like `기능 개선`, `코드 정리`, or `테스트 추가` without specifics.
- Avoid marketing language.
- Avoid English-only bullets unless the domain term is already English.

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
- The body bullets match the selected scope.
- No unsupported claim was added.
- Sensitive file contents were not exposed.
