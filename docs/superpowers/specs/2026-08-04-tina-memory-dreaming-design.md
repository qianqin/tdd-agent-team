# Tina Memory & Dreaming — Design

Date: 2026-08-04
Status: approved design, pre-implementation
Scope: part of the `tdd-agent-team` plugin

## Problem

Teammate subagents are stateless by design, but Tina (the orchestrator) currently is
too: every session she relearns the user's preferences, each machine's quirks, and
which orchestration decisions failed last time. She needs persistent long-term memory,
maintained the way most agent-memory systems do it — an offline "dreaming" pass that
periodically distills recent session transcripts into curated memory and prunes what
went stale.

## Prior art this design draws on

- **Letta/MemGPT sleep-time agents**: the working agent never edits memory; a separate
  offline process rewrites it ("raw context → learned context").
- **mem0**: per-candidate-fact LLM adjudication — ADD / UPDATE / DELETE / NOOP against
  similar existing memories; contradiction resolution is a prompt, not a rule engine.
- **Hermes harness**: hard size budgets with no auto-compaction (over-budget writes must
  consolidate), and a do-not-capture list (no transient errors, no "tool X doesn't
  work" claims, no task-progress logs).
- **Claude Code Auto Dream / community nightly-cron setups**: grep transcripts for
  signal, never read them end-to-end; guards (24h cursor, skip active sessions, lock);
  loud failure handling; hard line caps because memory files past ~200 lines are
  silently invisible.

## Decisions (from brainstorming)

1. Ships inside the `tdd-agent-team` plugin; the memory is **Tina's alone**. Teammate
   subagents stay stateless and never see memory files.
2. **Dream-only writer**: Tina only reads memory; the dream process is the sole writer.
   No journal, no in-session memory writes.
3. Dream input: **all sessions** under the workspace in the last 24h (team runs or not).
4. Trigger: manual `/tdd-agent-team:dreaming` command **plus** a daily local cron
   installed by a `setup` mode. (Transcripts are local, so cloud schedules can't work.)
5. Two kinds of memory files, split by scope (see layout below). Repo memory is
   committed; global memory lives in the (non-git) workspace root.

## Memory layout

The **workspace root** is wherever Claude is started — typically a plain folder (not a
git repo) containing one subfolder per repo. No path is ever hardcoded; everything is
resolved relative to the start folder at runtime.

### `<workspace-root>/memory.local.md` — global memory (one per machine)

```markdown
---
last_dream: 2026-08-04T03:12:00Z
---

# Tina's Memory (global)

<!-- Maintained by the dreaming skill. Do not edit by hand. Max 80 lines. -->

## User
- [2026-07-28] Prefers small tasks: rejects plans where a task touches >5 files

## Machine
- [2026-08-02] npm test needs NODE_OPTIONS=--max-old-space-size=4096 on this VM

## Orchestration lessons
- [2026-08-01] Review FAIL findings must be relayed verbatim, no softening
```

- Sections: `## User` (preferences, corrections, working style), `## Machine` (facts
  about this VM/host), `## Orchestration lessons` (cross-project lessons about running
  the team).
- Frontmatter `last_dream` is the dream's idempotency cursor — machine-local by nature,
  no separate state file.
- Budget: **80 lines** total.

### `<repo>/docs/memory.md` — per-repo memory (committed)

```markdown
# Tina's Memory (repo)

<!-- Maintained by the dreaming skill. Do not edit by hand. Max 100 lines. -->

## Project
- [2026-07-30] Deploys are verified against the staging compose file, not local

## Orchestration lessons
- [2026-07-29] Auth tasks always overlap in middleware/ — sequence them, never parallel
```

- Lives in `docs/` alongside the plugin's existing `DEVOPS.md` / `SPEC.md` / `OSS.md`
  convention. Committed as part of that repo.
- Sections: `## Project` (stable facts about this codebase not derivable from its
  files), `## Orchestration lessons` (repo-specific team-run lessons).
- Budget: **100 lines** total.

### Shared format rules

- One fact per bullet, prefixed with an absolute date: `- [YYYY-MM-DD] fact`. The dream
  converts relative time references to absolute dates.
- Budgets are hard: the dream must merge/prune until each file fits — never silently
  truncate (files past Claude Code's load limits become invisible).
- **Do-not-capture list** (adopted from Hermes): no task progress or completed-work
  logs; no transient errors that got resolved; no "tool/command X doesn't work" claims
  (they harden into refusals long after the problem is fixed); nothing derivable from
  the repo itself (code structure, git history — that's CLAUDE.md's job).

## Tina integration (changes to `skills/tdd-agent-team/SKILL.md`)

- **Setup step**: locate `memory.local.md` in the start folder or the nearest ancestor
  directory that has one (covers sessions started inside a repo subfolder), read it,
  and apply it; note the `last_dream` value loaded. When a task targets a repo, also
  read that repo's `docs/memory.md`.
- **Periodic reread** (long-lived sessions): before dispatching any task and whenever
  resuming work after user input, re-check `last_dream` in `memory.local.md` (one
  cheap 3-line read). If newer than what was loaded, a dream ran since — reread all
  relevant memory files before continuing. This is how a day-old session picks up last
  night's dream.
- **Read-only**: Tina never writes memory files. If the user contradicts a memory
  entry, the user wins for the session; tonight's dream sees the correction in the
  transcript and updates the file.
- **Stateless teammates**: memory is never pasted into dispatch prompts. If a fact is
  relevant to a task (e.g. an env quirk), Tina folds it into that task's task details
  as a plain instruction — the existing channel.

## The dreaming skill

New skill in the plugin: `skills/dreaming/SKILL.md`, invoked as
`/tdd-agent-team:dreaming`, args: `force`, `setup`, `setup remove`. Run it from the
workspace root; sessions that were started inside repo subfolders are still discovered
via the slug-prefix scan below.

### Algorithm

1. **Guards.** Read `last_dream` from `./memory.local.md`. If < 20h ago and not
   `force`, exit ("already dreamed today"). Cron adds a `flock` so runs never overlap.
2. **Discover sessions.** Transcript dirs are `~/.claude/projects/<slug>/` where slug =
   absolute start-path with `/` → `-`. The workspace root's slug is a string prefix of
   every repo-started session's slug, so scan **all dirs matching the root slug or
   `<root-slug>-*`**. Collect `*.jsonl` with mtime since `last_dream`, capped at the
   newest 20 (log anything skipped — no silent truncation). Exclude files modified in
   the last 30 minutes (probably a live session) and the dream's own session.
3. **Digest in parallel.** One stateless subagent per transcript. Each greps/jq's the
   JSONL for signal — user messages (asks, corrections, stated preferences),
   failure→resolution arcs, team-run markers (dispatches, PASS/FAIL verdicts, BLOCKED),
   compaction summaries — and never reads the file end-to-end (transcripts can be MBs).
   Returns ≤ 30 lines of candidate facts, each tagged with a proposed scope (user /
   machine / orchestration-global / repo:`<name>`) and the session date. Repo
   attribution comes from the transcript's `cwd` fields and worktree paths in the work.
4. **Consolidate.** With current memory files + all digests in context, adjudicate each
   candidate fact mem0-style: **ADD** (new), **UPDATE** (refines an existing entry —
   merge, keep newest date), **DELETE** (contradicted — newest wins), **NOOP**
   (redundant or on the do-not-capture list). Route by scope tag to the right file and
   section. Enforce budgets: merge similar entries first, then drop the oldest
   least-load-bearing ones.
5. **Write atomically.** Temp-file + rename per file, so a crash never leaves half a
   memory. Update `last_dream` last. For each changed `docs/memory.md` inside a git
   checkout: commit only that file (`chore(memory): dream YYYY-MM-DD`), never push.
6. **Report.** Final message = changelog per file (N added / M updated / K deleted) —
   visible when run manually and in the cron log.

First run creates missing memory files from templates — setup is "run it once".

### Cron setup (`setup` mode)

- Detects platform: crontab entry (Linux) or launchd plist (macOS). Default 03:00
  daily, per-machine minute jitter.
- Job: `cd <workspace-root> && flock -n <lockfile> claude -p "/tdd-agent-team:dreaming"
  --max-turns 40 >> ~/.claude/dream.log 2>&1`
- Verifies headless auth with one tiny `claude -p` test call; points the user at
  `claude setup-token` if it fails.
- `setup remove` uninstalls the job.

### Failure handling

- Errors leave memory files untouched (atomic writes) and `last_dream` unadvanced, so
  the next night retries automatically.
- `--max-turns` caps runaway cost; the log keeps per-run changelogs so silent failure
  is visible as a missing entry.

## Out of scope (YAGNI)

- No vector search / embedding retrieval — curated markdown within budget is enough at
  this scale.
- No hooks (SessionEnd capture etc.) — pure dream skill; can be added later if dream
  cost becomes an issue.
- No memory for teammate subagents.
- No importance scoring or time-decay math — staleness is handled by contradiction
  (newest wins) plus budget pressure.

## Verification (manual, since this is all skill/prompt files)

1. Run `/tdd-agent-team:dreaming force` on real last-24h sessions: digests produced,
   facts routed to the correct files/sections, budgets enforced, `last_dream` advanced.
2. Rerun without `force`: exits on the 20h guard.
3. Plant a contradiction across two sessions ("we use X", later "we switched to Y"):
   the dream keeps only the newer fact.
4. Overfill test: seed a memory file near its budget, dream with new facts, confirm
   merge/prune keeps it under budget with the newest facts intact.
5. `setup` on macOS: launchd job installed, fires, log shows a changelog; `setup
   remove` cleans up.
6. Long-lived session test: start a Tina session, run a forced dream from another
   terminal, send Tina a new message — she detects the newer `last_dream` and rereads.
