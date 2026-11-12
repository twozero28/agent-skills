# Spring Boot Conventions

An Agent Skill for predictable Java/Spring Boot implementation, testing, and review.

The design combines feature-first organization, domain-owned business rules, code-level CQRS, and meaningful UseCase orchestration. It does not mandate JPA, jOOQ, MyBatis, a particular Java/Spring version, or a new testing framework.

## Contents

```text
spring-boot-conventions/
├── SKILL.md
├── README.md
├── agents/
│   └── openai.yaml
└── references/
    ├── architecture.md
    ├── package-structure.md
    ├── naming.md
    ├── code-style.md
    ├── graphql-jpa.md
    ├── transactions.md
    ├── testing.md
    ├── sql-migrations.md
    ├── implementation-workflow.md
    ├── review-checklist.md
    └── source-notes.md
```

[SKILL.md](SKILL.md) is the entry point and reference router. Load detailed documents only when the task needs them. `agents/openai.yaml` contains optional Codex display metadata, not a second rule set. Use this README for installation and usage.

## Main decisions

| Concern | Convention |
|---|---|
| Organization | Feature first; create only necessary standard package categories |
| Single business operation | API -> Service -> Repo |
| Read operation | API -> QueryService -> QueryRepo |
| Composite workflow | API -> UseCase -> Services -> their Repos |
| Domain rules | Domain models/policies own invariants and transitions, regardless of complexity |
| GraphQL | Reuse business-entity types and relationships across list/detail/search |
| JPA | Entities may serve as domain models and GraphQL result objects without redundant copies |
| Transactions | Required Service boundaries plus a UseCase boundary when composition must be atomic |
| Inputs | Reuse equivalent Draft/Patch inputs; separate contracts only when semantics differ |
| Tests | Preserve the execution path being verified; keep fixtures and cleanup in test support |
| Project integration | Existing scoped instructions, formatters, test rules, and contracts take precedence |

The detailed rule owners are [Architecture](references/architecture.md), [GraphQL and JPA](references/graphql-jpa.md), [Package structure](references/package-structure.md), [Naming](references/naming.md), [Code style](references/code-style.md), [Transactions](references/transactions.md), [Testing](references/testing.md), and [SQL and migrations](references/sql-migrations.md).

## Local installation

Copy the **entire `spring-boot-conventions` folder**, not just `SKILL.md`, into the location supported by the agent tool. Check the tool's current documentation before installation; local paths do not imply that a hosted session can read local files.

| Tool | Project-local directory | Personal directory |
|---|---|---|
| Codex | `.agents/skills/spring-boot-conventions/` | `~/.agents/skills/spring-boot-conventions/` |
| Claude Code | `.claude/skills/spring-boot-conventions/` | `~/.claude/skills/spring-boot-conventions/` |

See [tool references](references/source-notes.md#agent-skill-format-and-tool-integration). Compare local modifications before replacing an existing copy. Avoid nesting a second folder of the same name. Resolve overlapping older skills, such as `java-conventions`, without deleting files automatically.

## Invocation examples

Codex:

```text
$spring-boot-conventions
Implement facility approval using the existing project structure and instructions.
```

Claude Code:

```text
/spring-boot-conventions Review the transaction boundaries in this change.
```

## Connect project instructions without duplicating the skill

Merge a short reference into the applicable `AGENTS.md` or `CLAUDE.md`; do not replace the existing file.

```markdown
## Java / Spring Boot

- Apply `spring-boot-conventions` when implementing, changing, testing, or reviewing Java/Spring Boot code.
- Read its entry point and only the references relevant to the task.
- Follow this project's scoped exceptions, formatter, test infrastructure, and required verification commands.
```

Keep project-specific decisions here, not copies of the skill's detailed rules. For example, preserve a project's test annotation names, supported infrastructure, fixture tools, TDD process, build commands, and completion requirements. Do not erase a project's test instructions to replace them with generic guidance.

## Usage notes

Respond in the user's language. Follow project conventions for comments, test names, and business text.

Adapt code examples to actual project types and versions. Do not add dependencies or production APIs merely to copy an example.
