# agent-skills

개인적으로 사용하는 에이전트 스킬을 여러 도구와 환경에서 재사용하기 위한 저장소입니다.

## 포함된 스킬

| 스킬 | 설명 |
| --- | --- |
| [`review-code-maintainability`](./skills/review-code-maintainability/README.md) | 코드 구조와 가독성, 책임 분리, 변경 비용을 중심으로 리뷰합니다. |
| [`commit-message-from-work`](./skills/commit-message-from-work/README.md) | 현재 작업을 사용자 언어의 Conventional Commit 메시지로 정리합니다. |

## 설치

```bash
npx skills add git@github.com:twozero28/agent-skills.git -s <스킬명>
```

## 저장소 구조

```text
agent-skills/
├── README.md
└── skills/
    ├── commit-message-from-work/
    │   ├── README.md
    │   ├── SKILL.md
    │   └── agents/
    │       └── openai.yaml
    └── review-code-maintainability/
        ├── README.md
        ├── SKILL.md
        └── agents/
            └── openai.yaml
```
