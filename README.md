# tdd-agent-team

A Claude Code skill that turns a feature spec into a gated, test-driven, multi-agent workflow.

**Tina Taskmaster** (your main session) breaks the spec into tasks with BDD scenarios, then dispatches specialized subagents — each a dedicated plugin agent type (shown under its own name in the UI) with minimal context (its role prompt + task details only):

| Agent | Role |
|---|---|
| billy-builder | Developer — strict RED/GREEN/REFACTOR TDD, one per parallel task |
| nick-picker | Code review — quality & spec compliance, reviews the diff |
| betty-bugsniff | QA — test quality, coverage, runs the suite; integration tests on main |
| sam-shields | Security review — attacker mindset, severity-graded, dependency CVE audit |
| daisy-deployer | DevOps — deploy verification with mandatory rollback safety |
| danny-digester | Dreaming — digests one session transcript into candidate memory facts |

Every task passes four gates: dev done → all three reviews PASS → deploy verified → integration tests green on main. Each task runs in its own git worktree; the main checkout never leaves `main`.

## Install

```
/plugin marketplace add qianqin/tdd-agent-team
/plugin install tdd-agent-team@qian-skills
```

Manual alternative: clone this repo, copy `skills/tdd-agent-team/` into `~/.claude/skills/`, and copy the files in `agents/` into `~/.claude/agents/`.

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

## Memory & dreaming

Tina has persistent long-term memory; teammates stay stateless. Tina never writes her
own memory — it is maintained exclusively by an offline "dreaming" pass, the way
[Letta's sleep-time agents](https://docs.letta.com/guides/agents/architectures/sleeptime/)
separate the working agent from the memory-writing agent. Each dream mines the last
24h of session transcripts for durable learnings (what you asked for, what you
corrected, what worked, what failed) and consolidates them mem0-style — each candidate
fact is ADDed, UPDATEd, DELETEd (contradicted: newest wins), or dropped — under hard
size budgets with no silent truncation, a discipline borrowed from the
[Hermes harness](https://github.com/NousResearch/hermes-agent) (as is its
do-not-capture list: no task progress, no transient errors, no stale "X doesn't work"
claims).

Memory files:

- `memory.local.md` in the folder you start Claude from — user preferences, machine
  facts, cross-project orchestration lessons (max 80 lines, personal to the machine)
- `docs/memory.md` inside each repo — project quirks and repo-specific lessons,
  committed with the repo (max 100 lines)

### Using it

```
/tdd-agent-team:dreaming              # dream now (no-op if dreamed in the last 20h)
/tdd-agent-team:dreaming force        # dream now regardless
/tdd-agent-team:dreaming setup        # install the daily 03:00 cron (launchd/crontab)
/tdd-agent-team:dreaming setup remove # uninstall the cron
```

Run it (and start Claude) from your workspace root — the folder that contains your
repos. The first dream creates the memory files; every dream ends with a changelog of
what it added, updated, and deleted, also written to `~/.claude/dream.log` when run by
cron. Tina rereads memory whenever a dream has run since she loaded it, so even a
days-old session picks up last night's learnings before dispatching new work.

## Layout

```
agents/                         # One agent definition per teammate (name + role prompt)
skills/tdd-agent-team/
├── SKILL.md                    # Tina's orchestration instructions
└── references/
    ├── workflow.md             # The gated per-task workflow (Tina only)
    └── templates.md            # /docs/ file templates
skills/dreaming/
├── SKILL.md                    # Daily memory consolidation ("dreaming") + cron setup
└── references/
    └── templates.md            # Memory file templates
```
