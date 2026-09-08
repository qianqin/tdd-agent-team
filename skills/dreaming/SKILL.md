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
  `## Orchestration lessons`. Max 100 lines. It is a committed file: it belongs on the
  repo's default branch, in the repo's PRIMARY working tree — never inside a
  `.worktrees/` task checkout, whose branch gets deleted after the merge.
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
(or bare `danny-digester`, whichever appears in your available agent types) per
transcript, all in one message. Each prompt contains ONLY: the transcript path, the
workspace root, and today's date. Each returns `SESSION:` header lines plus
`- scope | fact` candidate lines (scopes: `user`, `machine`, `orchestration`,
`repo:<name>`).

**STEP 4 — Consolidate.** Read the current global memory file and the `docs/memory.md`
of every repo named in the digests (repos without one yet get the template). Resolve
each repo to its primary working tree first — `git -C <path> worktree list --porcelain
| head -1` gives `worktree <primary path>` — and read/write `docs/memory.md` only
there, even when the digest's session ran inside a task worktree. For each candidate
fact decide exactly one of:

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
leaves the cursor unadvanced so the next run retries.

**STEP 5b — Commit and push each changed `docs/memory.md`.** A dreamt repo memory that
stays in the working tree is worse than no dream at all: the default branch never gets
the facts, and every task worktree branched off it starts stale. So for each changed
`docs/memory.md` in a git checkout (always the primary working tree from STEP 4), in
this order:

1. `git -C <repo> rev-parse --abbrev-ref HEAD`. The default branch is the target of
   `origin/HEAD` (`git -C <repo> symbolic-ref --short refs/remotes/origin/HEAD`), else
   `main`, else `master`. If HEAD is NOT the default branch, do NOT commit — leave the
   file written, and report it as pending with the exact commands, so the user lands it
   themselves once they are back on the default branch.
2. Bring the local branch up to date first, when it has an upstream: `git -C <repo>
   fetch origin && git -C <repo> pull --ff-only`. The memory commit then sits on the
   current tip, the push below is a plain fast-forward, and any worktree cut from this
   checkout afterwards starts current. If the pull is not a fast-forward, skip the
   commit and report it as pending — never merge or reset someone else's work to land a
   memory file.
3. `git -C <repo> add docs/memory.md`
4. `git -C <repo> commit -m "chore(memory): dream <YYYY-MM-DD>" -- docs/memory.md` —
   the `--` pathspec keeps anything else the user had staged out of the commit.
5. Push, if the branch has an upstream: `git -C <repo> push`. On a non-fast-forward
   rejection, `git -C <repo> pull --rebase` and push once more. No upstream, or a second
   failure → stop there. If the repo's memory or the user says pushes to `main` are not
   allowed (protected branch, PR-only, or a CI/CD pipeline that must not be triggered by
   a docs-only commit), skip the push, leave the commit local, and say so — this is the
   one case where "committed, not pushed" is correct rather than a failure.

Never `git add -A`, never amend, never force-push, never touch a branch other than the
default one. On any git error: stop at that step, keep the written file, and name the
failure in the report — a skipped commit or an unpushed commit is a headline, never a
silent skip.

**STEP 6 — Report.** Final message = a changelog: per file, counts of added / updated /
deleted entries (or "unchanged"), plus sessions processed and skipped. For every changed
`docs/memory.md`, state its git outcome explicitly — `committed + pushed`,
`committed, not pushed (<reason>)`, or `written, not committed (<reason>)` followed by
the commands to finish it.

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

5. The headless run has to be able to land STEP 5b's commit and push unattended, so
   check that `Bash(git add:*)`, `Bash(git commit:*)` and `Bash(git push:*)` are allowed
   in the user's settings (`~/.claude/settings.json` `permissions.allow`); if they are
   not, tell the user which ones to add — without them the cron dreams and then leaves
   every repo memory uncommitted.
6. Confirm to the user: schedule, log location (`~/.claude/dream.log`), and that
   `setup remove` uninstalls.

## `setup remove`

macOS: `launchctl unload ~/Library/LaunchAgents/com.tdd-agent-team.dreaming.plist` and
delete the plist. Linux: remove the dreaming line from `crontab -l`. Confirm removal.
