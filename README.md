# tdd-agent-team

A Claude Code skill that turns a feature spec into a gated, test-driven, multi-agent workflow.

**Tina Taskmaster** (your main session) breaks the spec into tasks with BDD scenarios, then dispatches specialized subagents — each with minimal context (its role file + task details only):

| Agent | Role |
|---|---|
| billy-builder-\<task\> | Developer — strict RED/GREEN/REFACTOR TDD, one per parallel task |
| nick-picker | Code review — quality & spec compliance, reviews the diff |
| betty-bugsniff | QA — test quality, coverage, runs the suite; integration tests on main |
| sam-shields | Security review — attacker mindset, severity-graded, dependency CVE audit |
| daisy-deployer | DevOps — deploy verification with mandatory rollback safety |

Every task passes four gates: dev done → all three reviews PASS → deploy verified → integration tests green on main. Each task runs in its own git worktree; the main checkout never leaves `main`.

## Install

```
/plugin marketplace add qianqin/tdd-agent-team
/plugin install tdd-agent-team@qian-skills
```

Manual alternative: clone this repo and copy `skills/tdd-agent-team/` into `~/.claude/skills/`.

## Use

Say things like "implement this spec with a team", "spawn a TDD team", or invoke directly:

```
/tdd-agent-team:tdd-agent-team
```

Tina will read your spec, propose a task plan with BDD scenarios, and wait for your approval before dispatching agents.

### Optional project docs

- `/docs/DEVOPS.md` — deployment config; required for the deploy gate (templates included in the skill)
- `/docs/OSS.md` — dependency license policy, enforced by the security reviewer
- `/docs/SPEC.md` — architecture reference used by code review

## Layout

```
skills/tdd-agent-team/
├── SKILL.md                    # Tina's orchestration instructions
└── references/
    ├── workflow.md             # The gated per-task workflow (Tina only)
    ├── templates.md            # /docs/ file templates
    └── agents/                 # One minimal role file per subagent
```
