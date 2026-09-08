# tdd-agent-team

A Claude Code skill that turns a feature spec into a gated, test-driven, multi-agent workflow.

**Tina Taskmaster** (your main session) breaks the spec into tasks with BDD scenarios, then dispatches specialized subagents — each a dedicated plugin agent type (shown under its own name in the UI) with minimal context (its role prompt + task details only):

| Agent | Role |
|---|---|
| billy-builder | Generic developer — strict RED/GREEN/REFACTOR TDD, fallback when no domain fits |
| fiona-frontend | Frontend developer — webdesign focus, designs while building |
| benny-backend | Backend developer — APIs, data, migrations |
| frank-firmware | Firmware developer — embedded constraints, HAL-tested |
| mandy-mobile | Mobile developer — platform conventions, offline-first |
| nick-picker | Code review — quality & spec compliance, reviews the diff |
| betty-bugsniff | QA — test quality, coverage, runs the suite; integration tests on main |
| sam-shields | Security review — attacker mindset, severity-graded, dependency CVE audit |
| daisy-deployer | DevOps — deploy verification with mandatory rollback safety |
| polly-pixels | Design review — inspects the rendered UI of frontend tasks |
| wally-wordsmith | Docs writer — updates affected docs after reviews pass, pre-merge |
| danny-digester | Dreaming — digests one session transcript into candidate memory facts |

Every task passes five gates: dev done → all three reviews PASS → deploy check verified → integration tests green on the merged `main` → released: pushed, pipeline green, production healthy. Devs run only their task's scope plus the build; the full suite runs twice, both times by QA — in the worktree at the review gate, and on the locally merged `main` just before it is pushed. The push to `origin/main` is the last step of a task, because that is what triggers CI/CD — daisy-deployer makes it and watches the pipeline through to a verified-healthy production, rolling back if it isn't. Each task runs in its own git worktree; the main checkout never leaves `main`. Tina arms an hourly heartbeat for the run: if a teammate dies on a rate limit or a timeout, the next tick reads the task list and re-dispatches it from the step it stalled on, one agent at a time. It renews before its 7-day expiry and cancels itself when the run is done.

When several Tinas run at once they coordinate by events, not polling: a claim goes out when a task starts and a release when it finishes, each naming the branch and the paths it owns. A Tina who sees a claim overlapping her own work can ask the holder to hand it over, with her reasoning and what her team is already doing — and the holder decides. Nobody's running dev is ever cancelled by someone else's request.

That makes it worth running teams with different focuses — one on a full feature, one on the fast lane for typos, copy and layout nudges. A tiny edit landing on the feature team gets offered to the fast-lane team instead of stalling behind five gates,. Only queued tasks ever move between teams; a dispatched dev always finishes its own.

Integration assumes other teams share the repo: branches are cut from a freshly fetched `origin/main`, devs rebase onto it (never merge `main` in), and Tina lands the work with `git merge --ff-only` plus an immediate `git push origin main` — no PR. Say so in memory or in the session if your repo uses PRs, merge commits, or a protected `main`, and the team follows that instead.

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
  committed with the repo (max 100 lines). The dream commits and pushes it on the repo's
  default branch, so task worktrees branched off it are never stale; if the repo is on
  another branch, or the push fails, the dream says so instead of leaving it silently
  behind.

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
