# Tina Memory & Dreaming Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give Tina (the tdd-agent-team orchestrator) persistent long-term memory maintained by a daily "dreaming" pass that distills recent session transcripts into budgeted memory files.

**Architecture:** Tina stays a read-only consumer of two kinds of memory files — a global `memory.local.md` in the workspace root and a committed `docs/memory.md` per repo. A new `dreaming` skill (run manually or via a cron its `setup` mode installs) discovers last-24h transcripts, digests each in a parallel `danny-digester` plugin agent (grep-first, never full reads), then consolidates candidate facts with ADD/UPDATE/DELETE/NOOP adjudication under hard line budgets and writes atomically.

**Tech Stack:** Claude Code plugin (markdown skills + plugin agents), `jq`/`grep` for transcript mining, `claude -p` headless runs, crontab (Linux) / launchd (macOS).

**Spec:** `docs/superpowers/specs/2026-08-04-tina-memory-dreaming-design.md` — read it before starting any task.

## Global Constraints

- The dreaming skill is the ONLY writer of memory files. Tina and all other agents are read-only.
- Global memory `memory.local.md`: max **80 lines**, sections `## User`, `## Machine`, `## Orchestration lessons`, frontmatter `last_dream` (ISO 8601 UTC).
- Repo memory `docs/memory.md`: max **100 lines**, sections `## Project`, `## Orchestration lessons`.
- Entry format everywhere: `- [YYYY-MM-DD] fact` — one fact per line, absolute dates only.
- Do-not-capture list (verbatim, appears in both the digester and the dreaming skill): task progress or completed-work logs; transient errors that got resolved; "tool/command X doesn't work" claims; anything derivable from the repo itself (code structure, git history, CLAUDE.md content); secrets or credentials.
- Never hardcode any workspace path (e.g. `~/projects`) — always resolve from the folder the session started in.
- Plugin naming conventions: agents live in `agents/<name>.md` with `name`/`description` frontmatter; skills live in `skills/<name>/SKILL.md`; teammate names are playful alliterations.
- Transcripts are mined with `jq`/`grep`/`head` — never read a `.jsonl` end-to-end into context.

---

### Task 1: Memory file templates

**Files:**
- Create: `skills/dreaming/references/templates.md`

**Interfaces:**
- Produces: the two exact memory-file templates that Task 3's skill instantiates when files are missing. Section names and budgets here are the canonical definitions.

- [ ] **Step 1: Write the templates file**

Create `skills/dreaming/references/templates.md` with exactly this content (the ISO timestamp is a placeholder the dreamer replaces at creation time):

````markdown
# Memory File Templates

Instantiate these when a memory file is missing. Replace `<NOW-ISO-8601-UTC>` with the
current UTC time (e.g. `2026-08-04T03:12:00Z`). Keep the HTML comments — they are the
in-file contract.

## Template: global memory — `<workspace-root>/memory.local.md`

```markdown
---
last_dream: <NOW-ISO-8601-UTC>
---

# Tina's Memory (global)

<!-- Maintained by the dreaming skill of the tdd-agent-team plugin. Do not edit by hand. -->
<!-- Budget: 80 lines total. One fact per bullet: - [YYYY-MM-DD] fact -->

## User

## Machine

## Orchestration lessons
```

## Template: repo memory — `<repo>/docs/memory.md`

```markdown
# Tina's Memory (repo)

<!-- Maintained by the dreaming skill of the tdd-agent-team plugin. Do not edit by hand. -->
<!-- Budget: 100 lines total. One fact per bullet: - [YYYY-MM-DD] fact -->

## Project

## Orchestration lessons
```
````

- [ ] **Step 2: Verify structure**

Run: `grep -c '^## Template' skills/dreaming/references/templates.md`
Expected: `2`

- [ ] **Step 3: Commit**

```bash
git add skills/dreaming/references/templates.md
git commit -m "feat(dreaming): memory file templates"
```

---

### Task 2: The danny-digester plugin agent

**Files:**
- Create: `agents/danny-digester.md`

**Interfaces:**
- Consumes: a dispatch prompt from the dreaming skill containing: absolute transcript path, absolute workspace root, today's date.
- Produces: a final response in the exact `SESSION:` line format below — Task 3's consolidation step parses it.

- [ ] **Step 1: Write the agent file**

Create `agents/danny-digester.md` with exactly this content:

````markdown
---
name: danny-digester
description: Transcript digester for the dreaming process — mines one Claude Code session transcript (.jsonl) for durable learnings and returns scope-tagged candidate facts. Dispatched by the tdd-agent-team dreaming skill; not for general use.
---

# Transcript Digester — Danny Digester

You digest ONE session transcript for Tina's dream process. Your dispatch prompt gives
you: the transcript path, the workspace root, and today's date. Your final response is
parsed by the dreamer — return ONLY the format below, no commentary.

## Rules

- NEVER read the .jsonl end-to-end — transcripts can be megabytes. Mine with
  `jq`/`grep`/`head`/`wc`, capping every extraction.
- A transcript line is one JSON object. Relevant fields: `type`
  (user/assistant/system/summary), `message.content` (string, or array of blocks with
  `.type`/`.text`), `cwd`, `timestamp`, `gitBranch`.
- Repo attribution: a fact belongs to `repo:<name>` when the work happened inside
  `<workspace-root>/<name>` — use `cwd` values and worktree paths in the conversation.
  `<name>` is the first path segment relative to the workspace root.

## Mining recipe (adapt commands if a field shape differs)

1. Session metadata: `head -c 2000 <file> | jq -r '.cwd, .timestamp'` — session date and
   starting folder.
2. What the user said (asks, corrections, preferences):
   `jq -r 'select(.type=="user") | .message.content | if type=="string" then . else (.[] | select(.type?=="text") | .text) end' <file> | head -c 20000`
   (This skips tool_result blocks automatically — they have no `text` field.)
3. Outcome markers (team runs, failures, resolutions):
   `jq -r 'select(.type=="assistant") | .message.content[]? | select(.type?=="text") | .text' <file> | grep -iE 'PASS|FAIL|BLOCKED|error|dispatch|worktree|rolled back' | head -50`
4. Compaction summaries: `jq -r 'select(.type=="summary") | .summary' <file>`

## What counts as a candidate fact

- `user` — stated preferences, corrections ("don't X", "always Y"), working style
- `machine` — facts about THIS machine: tool quirks, resource limits, local paths
- `repo:<name>` — stable facts about that repo: env realities, recurring pitfalls,
  repo-specific orchestration lessons (task splits that failed, review patterns)
- `orchestration` — cross-project lessons about running the team (task sizing, gate
  behavior, how findings should be relayed)

Do NOT capture: task progress or completed-work logs; transient errors that got
resolved; "tool/command X doesn't work" claims; anything derivable from the repo itself
(code structure, git history, CLAUDE.md content); secrets or credentials.

## Final response format (max 30 lines total)

```
SESSION: <transcript filename> DATE: <YYYY-MM-DD from timestamps>
- user | <fact>
- machine | <fact>
- repo:<name> | <fact>
- orchestration | <fact>
```

One `- scope | fact` line per fact, most important first. If nothing durable was
learned, return exactly: `SESSION: <transcript filename> — no durable learnings`
````

- [ ] **Step 2: Verify frontmatter and format contract**

Run: `head -4 agents/danny-digester.md && grep -c 'SESSION:' agents/danny-digester.md`
Expected: frontmatter opens with `---` and `name: danny-digester`; grep count ≥ 2.

- [ ] **Step 3: Commit**

```bash
git add agents/danny-digester.md
git commit -m "feat(dreaming): danny-digester transcript digest agent"
```

---

### Task 3: The dreaming skill

**Files:**
- Create: `skills/dreaming/SKILL.md`

**Interfaces:**
- Consumes: `skills/dreaming/references/templates.md` (Task 1), agent type `tdd-agent-team:danny-digester` and its `SESSION:` output format (Task 2).
- Produces: the `/tdd-agent-team:dreaming` command with args `force`, `setup`, `setup remove`; memory files in the exact template format; `last_dream` cursor semantics that Task 4's reread rule depends on.

- [ ] **Step 1: Write the skill file**

Create `skills/dreaming/SKILL.md` with exactly this content:

````markdown
---
name: dreaming
description: "Use when the user asks to dream, consolidate Tina's memory, run memory consolidation, or set up the dreaming cron — distills recent Claude Code session transcripts into Tina's long-term memory files. Also runs headlessly via the daily cron."
---

# Dreaming — Tina's Memory Consolidation

You are Tina's dream process: the ONLY writer of her memory files. The workspace root
is the folder this session started in. Args: `force` (skip the 20h guard), `setup`
(install the daily cron), `setup remove` (uninstall it). No args = a normal dream run.

`${CLAUDE_SKILL_DIR}` is substituted with this skill's directory when the skill loads;
if it appears unsubstituted, use the base directory shown when the skill was invoked.

## Memory files

- `./memory.local.md` — global. Sections `## User`, `## Machine`,
  `## Orchestration lessons`. Max 80 lines. Frontmatter `last_dream` (ISO 8601 UTC) is
  the idempotency cursor.
- `<repo>/docs/memory.md` — one per repo subfolder. Sections `## Project`,
  `## Orchestration lessons`. Max 100 lines.
- Entries: `- [YYYY-MM-DD] fact` — one fact per line, absolute dates only.
- Missing files are created from `${CLAUDE_SKILL_DIR}/references/templates.md`.

## Dream run (no args, or `force`)

**STEP 1 — Guard.** Read `last_dream` from `./memory.local.md`. If it is less than 20
hours ago and `force` was not given: report "Already dreamed at <last_dream>; use
'force' to dream again." and STOP.

**STEP 2 — Discover transcripts.** Session transcripts live in
`~/.claude/projects/<slug>/*.jsonl` where slug = the session's start path with `/`
replaced by `-`. Compute the workspace root's slug and collect candidate files from the
directory matching it exactly AND every directory matching `<slug>-*` (sessions started
inside repo subfolders). Filter:

- mtime newer than `last_dream` (no cursor yet → newer than 24 hours ago), e.g.
  `find <dirs> -name '*.jsonl' -newermt '<cutoff>'`
- EXCLUDE files modified in the last 30 minutes — they are probably live sessions (this
  also excludes this dream's own transcript)
- Cap at the 20 newest by mtime; if you drop any, say how many were skipped — never
  truncate silently.

If nothing remains: report "No new sessions to dream about." and STOP (do not advance
`last_dream`).

**STEP 3 — Digest in parallel.** Dispatch one `tdd-agent-team:danny-digester` subagent
per transcript, all in one message. Each prompt contains ONLY: the transcript path, the
workspace root, and today's date. Each returns `SESSION:` header lines plus
`- scope | fact` candidate lines (scopes: `user`, `machine`, `orchestration`,
`repo:<name>`).

**STEP 4 — Consolidate.** Read the current global memory file and the `docs/memory.md`
of every repo named in the digests (repos without one yet get the template). For each
candidate fact decide exactly one of:

- **ADD** — genuinely new. Route by scope: `user`/`machine`/`orchestration` → the
  matching global section; `repo:<name>` → that repo's `## Project` or repo-specific
  `## Orchestration lessons`.
- **UPDATE** — refines an existing entry: merge into one line, keep the newest date.
- **DELETE** — contradicts an existing entry: newest wins, remove the old line.
- **NOOP** — redundant, trivial, or on the do-not-capture list.

Also: convert every relative time reference to an absolute `[YYYY-MM-DD]`; drop
candidates matching the do-not-capture list: task progress or completed-work logs;
transient errors that got resolved; "tool/command X doesn't work" claims; anything
derivable from the repo itself (code structure, git history, CLAUDE.md content);
secrets or credentials.

**Budgets are hard.** If a file would exceed its cap (80 global / 100 repo), first
merge similar entries, then drop the oldest least-load-bearing ones until it fits.
Never leave a file over budget — oversized memory gets silently truncated at load time.

**STEP 5 — Write atomically.** For each changed file: write the full new content to a
temp file in the same directory, then `mv` it over the original. Write
`memory.local.md` LAST, with `last_dream` set to now (UTC) — an earlier crash then
leaves the cursor unadvanced so the next run retries. For each changed `docs/memory.md`
inside a git checkout: `git -C <repo> add docs/memory.md && git -C <repo> commit -m
"chore(memory): dream <YYYY-MM-DD>"` — commit only that file, NEVER push, skip on any
git error.

**STEP 6 — Report.** Final message = a changelog: per file, counts of added / updated /
deleted entries (or "unchanged"), plus sessions processed and skipped.

## `setup` — install the daily cron

1. Verify headless auth: run `claude -p "Reply with OK" --max-turns 1`. If it fails,
   tell the user to run `claude setup-token` and STOP.
2. Pick a random minute M (0–59). Let ROOT = the absolute workspace root.
3. macOS (`uname` = Darwin): write
   `~/Library/LaunchAgents/com.tdd-agent-team.dreaming.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.tdd-agent-team.dreaming</string>
  <key>ProgramArguments</key><array>
    <string>/bin/bash</string><string>-lc</string>
    <string>cd ROOT &amp;&amp; claude -p "/tdd-agent-team:dreaming" --max-turns 40 >> ~/.claude/dream.log 2>&amp;1</string>
  </array>
  <key>StartCalendarInterval</key><dict>
    <key>Hour</key><integer>3</integer><key>Minute</key><integer>M</integer>
  </dict>
</dict></plist>
```

   then `launchctl load ~/Library/LaunchAgents/com.tdd-agent-team.dreaming.plist`.
   (launchd never runs two instances of one label, and the 20h guard makes reruns
   no-ops — no extra locking needed.)
4. Linux: append to `crontab -l`:

```
M 3 * * * cd ROOT && flock -n /tmp/tdd-dreaming.lock claude -p "/tdd-agent-team:dreaming" --max-turns 40 >> ~/.claude/dream.log 2>&1
```

5. Confirm to the user: schedule, log location (`~/.claude/dream.log`), and that
   `setup remove` uninstalls.

## `setup remove`

macOS: `launchctl unload ~/Library/LaunchAgents/com.tdd-agent-team.dreaming.plist` and
delete the plist. Linux: remove the dreaming line from `crontab -l`. Confirm removal.
````

- [ ] **Step 2: Verify structure**

Run: `head -4 skills/dreaming/SKILL.md && grep -c '^\*\*STEP' skills/dreaming/SKILL.md`
Expected: frontmatter with `name: dreaming`; grep count `6`.

- [ ] **Step 3: Commit**

```bash
git add skills/dreaming/SKILL.md
git commit -m "feat(dreaming): dream skill — transcript discovery, digestion, consolidation, cron setup"
```

---

### Task 4: Tina reads memory (SKILL.md changes)

**Files:**
- Modify: `skills/tdd-agent-team/SKILL.md` (Setup list and Tina's Rules section)

**Interfaces:**
- Consumes: memory file locations, section names, and the `last_dream` cursor exactly as defined in Task 3.

- [ ] **Step 1: Add the memory setup step**

In `skills/tdd-agent-team/SKILL.md`, the Setup list currently begins:

```markdown
## Setup

1. Read `${CLAUDE_SKILL_DIR}/references/workflow.md` — only Tina reads it, never paste it into a dispatch prompt
2. Understand the task (read spec files, ask the user clarifying questions)
```

Insert a new step 2 (renumber the rest):

```markdown
2. Load memory: find `memory.local.md` in the start folder or the nearest ancestor that has one; read it and note its `last_dream` value. When the task targets a repo, also read that repo's `docs/memory.md`. Apply what you learn to planning and task details. Memory files are READ-ONLY for you — only the dreaming skill writes them.
```

- [ ] **Step 2: Add the reread and stateless rules**

Append to the `## Tina's Rules` list:

```markdown
- Memory reread: before dispatching any task and whenever you resume work after user input, re-check `last_dream` in `memory.local.md` (e.g. `head -3`). If it is newer than the value you loaded, a dream ran since — reread all relevant memory files before continuing.
- Never paste memory files into dispatch prompts — teammates stay stateless. If a memory fact matters for a task (e.g. an env quirk), fold it into that task's task details as a plain instruction.
- If the user contradicts a memory entry, the user wins for this session — do not edit the file; tonight's dream will pick the correction up from the transcript.
```

- [ ] **Step 3: Verify**

Run: `grep -n 'memory' skills/tdd-agent-team/SKILL.md`
Expected: hits in the Setup list (step 2) and three new lines in Tina's Rules; setup steps numbered 1–6 without gaps.

- [ ] **Step 4: Commit**

```bash
git add skills/tdd-agent-team/SKILL.md
git commit -m "feat(tdd-agent-team): Tina loads and periodically rereads dream memory"
```

---

### Task 5: Documentation

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a Memory & dreaming section**

After the `### Optional project docs` section in `README.md`, insert:

```markdown
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
```

- [ ] **Step 2: Update the layout tree**

Replace the contents of the Layout section's fenced code block (keeping its existing ``` fences) with this tree:

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

Also add `danny-digester` to the agent table:

```markdown
| danny-digester | Dreaming — digests one session transcript into candidate memory facts |
```

- [ ] **Step 3: Verify**

Run: `grep -c 'dreaming' README.md`
Expected: ≥ 3.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: memory & dreaming"
```

---

### Task 6: End-to-end verification with synthetic transcripts

**Files:**
- Create (temporary, deleted at the end): `/private/tmp/dream-e2e/` workspace and fixture transcript dirs under `~/.claude/projects/`

This exercises the real pipeline with `claude -p`. It costs a few real model calls.

- [ ] **Step 1: Build the fixture workspace**

```bash
mkdir -p /private/tmp/dream-e2e/alpha/docs
git -C /private/tmp/dream-e2e/alpha init -q
mkdir -p ~/.claude/projects/-private-tmp-dream-e2e ~/.claude/projects/-private-tmp-dream-e2e-alpha
```

- [ ] **Step 2: Write two synthetic transcripts (with a contradiction)**

```bash
cat > ~/.claude/projects/-private-tmp-dream-e2e/s1.jsonl <<'EOF'
{"type":"user","message":{"role":"user","content":"please always propose at most 3 parallel tasks, more is too noisy for me"},"cwd":"/private/tmp/dream-e2e","timestamp":"2026-08-03T09:00:00Z","sessionId":"s1"}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"Understood — capping plans at 3 parallel tasks."}]},"cwd":"/private/tmp/dream-e2e","timestamp":"2026-08-03T09:00:10Z","sessionId":"s1"}
{"type":"user","message":{"role":"user","content":"alpha uses npm for everything, tests run with npm test"},"cwd":"/private/tmp/dream-e2e","timestamp":"2026-08-03T09:01:00Z","sessionId":"s1"}
EOF
cat > ~/.claude/projects/-private-tmp-dream-e2e-alpha/s2.jsonl <<'EOF'
{"type":"user","message":{"role":"user","content":"we switched alpha from npm to pnpm today, use pnpm test from now on"},"cwd":"/private/tmp/dream-e2e/alpha","timestamp":"2026-08-03T15:00:00Z","sessionId":"s2"}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"Noted — alpha now uses pnpm; pnpm test verified PASS."}]},"cwd":"/private/tmp/dream-e2e/alpha","timestamp":"2026-08-03T15:00:10Z","sessionId":"s2"}
EOF
touch -t "$(date -v-2H +%Y%m%d%H%M)" ~/.claude/projects/-private-tmp-dream-e2e/s1.jsonl ~/.claude/projects/-private-tmp-dream-e2e-alpha/s2.jsonl
```

The `touch -t` backdates mtime 2 hours (macOS syntax) so the 30-minute live-session
exclusion doesn't skip the fixtures.

- [ ] **Step 3: Run the dream**

```bash
cd /private/tmp/dream-e2e && claude -p "/tdd-agent-team:dreaming force" --max-turns 40
```

- [ ] **Step 4: Verify outcomes**

Check each; any failure → fix the responsible skill/agent file and rerun Step 3:

```bash
head -3 /private/tmp/dream-e2e/memory.local.md          # frontmatter with fresh last_dream
grep -A3 '## User' /private/tmp/dream-e2e/memory.local.md  # "- [2026-08-03] ...3 parallel tasks..." style entry
cat /private/tmp/dream-e2e/alpha/docs/memory.md          # contains pnpm fact; NO npm fact (newest wins)
wc -l /private/tmp/dream-e2e/memory.local.md /private/tmp/dream-e2e/alpha/docs/memory.md  # ≤80 / ≤100
git -C /private/tmp/dream-e2e/alpha log --oneline        # one "chore(memory): dream ..." commit
```

- [ ] **Step 5: Verify the guard**

```bash
cd /private/tmp/dream-e2e && claude -p "/tdd-agent-team:dreaming" --max-turns 5
```

Expected output: "Already dreamed at <timestamp>..." and no file changes.

- [ ] **Step 6: Clean up fixtures**

```bash
rm -rf /private/tmp/dream-e2e ~/.claude/projects/-private-tmp-dream-e2e ~/.claude/projects/-private-tmp-dream-e2e-alpha
```

- [ ] **Step 7: Commit any fixes made during verification**

```bash
git add -A skills/ agents/ && git commit -m "fix(dreaming): adjustments from e2e verification" || echo "nothing to fix"
```

---

## Remaining manual checks (after Task 6, on the real workspace)

The spec's verification list has three items Task 6's headless harness can't cover; run them once during real use:

1. **Overfill**: when a memory file approaches its budget, confirm a dream merges/prunes it back under the cap with the newest facts intact.
2. **Cron install**: run `/tdd-agent-team:dreaming setup` on the real workspace; confirm the launchd job fires at 03:xx and `~/.claude/dream.log` gets a changelog; then decide whether to keep or `setup remove`.
3. **Long-lived session reread**: with a Tina session open, run a forced dream from another terminal, send Tina a message — she must detect the newer `last_dream` and reread before dispatching.

## Not covered on purpose (spec's YAGNI list)

No vector search, no hooks, no teammate memory, no importance scoring / time-decay math. Cron `setup` on Linux and the macOS launchd path share one code path in the skill; only macOS is manually verified in this environment (Darwin) — the crontab line is reviewed, not executed.
