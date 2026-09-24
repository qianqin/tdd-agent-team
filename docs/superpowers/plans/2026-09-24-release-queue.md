# Release Queue Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Concurrent Tinas on one machine release through a FIFO file queue in the git common dir. Only the head integrates, tests and pushes, and waiting teams pre-test on the stack ahead of them.

**Architecture:** This is a docs-only plugin. Behaviour is whatever the markdown tells Tina and her agents to do. `references/workflow.md` gains the full protocol (a new Release Queue section plus edits to the heartbeat, coordination, fast lane, test allocation and STEP 5–13). `SKILL.md` points at it. betty gains a pre-test mode and a scoped Gate 2, daisy gains a head-of-queue check before pushing, the DEVOPS template gains `parallel suites`, and the README gets a paragraph.

**Tech Stack:** Claude Code plugin markdown (skills + agents). Verification is grep over the edited files.

**Spec:** `docs/superpowers/specs/2026-09-23-release-queue-design.md`. Read it before starting. It holds the reasons behind every rule below.

## Global Constraints

- Queue path is exactly `$(git rev-parse --git-common-dir)/release-queue/` with `tickets/` and `abandoned/`, and is never in the working tree.
- Ticket keys are exactly: `team`, `session`, `task`, `branch`, `worktree`, `head`, `state`, `tested_tree`, `spec_on`, `updated`, `nudged`.
- States are exactly: `waiting` · `pretesting` · `ready` · `releasing` · `releasing:suite`.
- Messages are exactly `TEAM-QUEUE` (broadcast to every Tina) and `TEAM-QUEUE-NUDGE` (direct to the head's session).
- Train depth is 2 (positions #2 and #3 pre-test). Nudge after 5 minutes, abandon 15 minutes after the nudge.
- The task tag is exactly `full-suite-at-gate2: yes|no`. The DEVOPS setting is exactly `parallel suites: yes|no`, default `yes`.
- Agent final-response first lines stay `PASS`/`FAIL` (Tina parses them). betty adds a `tested tree: <hash>` line for any full run.
- When alone (no peer Tina), STEP 9–12 behave as today and no ticket is ever written.
- Do not bump `.claude-plugin/plugin.json`. Releasing the plugin is a separate, user-approved step.

## Review Focus

1. **macOS timestamps.** `date +%s%N` prints a literal `N` on macOS, which breaks queue order. The plan uses `python3 -c 'import time; print(time.time_ns())'` (Task 1 check greps for it and for the absence of `%N`).
2. **Half-written tickets.** A peer reading while the owner writes must never see a partial file. Writes go through `ticket.tmp` + `mv` (Task 1 check greps for `ticket.tmp`).
3. **A successful release ahead must not force a re-run behind.** When the team ahead leaves after a clean release, `spec_on` mismatches, but the rebuilt stack tree still equals `tested_tree`, so there's no suite run (Task 1 check greps for the tree-comparison wording).
4. **An idle #2 Tina has no clock.** The nudge and abandon windows need one-shot crons, and `nudged` must be recorded in her own ticket so the heartbeat can continue (Task 1 check greps for `recurring: false` and `nudged`).
5. **Stopping a pre-test must actually stop the suite.** betty runs it in the foreground under `nice -n 19` so that stopping her stops it (Task 3 check greps for `foreground`).

---

### Task 1: Release queue protocol in workflow.md

**Files:**
- Modify: `skills/tdd-agent-team/references/workflow.md` (Heartbeat section ~L44-48 and ~L73-74; coordination table ~L98-105; new section before `## Task Classes`; fast-lane block ~L187-201; `## Who Runs Which Tests` ~L223-241; workflow STEP block ~L278-338)

**Interfaces:**
- Produces: the names Tasks 2–5 reference: section `## Release Queue — One Release at a Time`, subsection `### Heartbeat Check`, `STEP 8.5`, ticket keys/states, `TEAM-QUEUE`, `TEAM-QUEUE-NUDGE`, `full-suite-at-gate2`, `parallel suites`, betty "pre-test mode", `tested tree: <hash>`.

- [ ] **Step 1: Heartbeat prompt includes the queue check.** Replace

```
  still alive (`ListAgents`, task list state, the agent's result). Re-dispatch anything
  that died — rate limit, timeout, error — from the STEP it was at. Touch nothing else.
```

with

```
  still alive (`ListAgents`, task list state, the agent's result). Re-dispatch anything
  that died — rate limit, timeout, error — from the STEP it was at. Then run the release
  queue's Heartbeat Check (see Release Queue) when this team has a ticket or leftover
  tickets. Touch nothing else.
```

- [ ] **Step 2: Heartbeat rule names its one exception.** Replace

```
- The heartbeat is a checker, not a second orchestrator. It resumes what stalled; it
  never starts a new task, never advances a gate, and never pushes.
```

with

```
- The heartbeat is a checker, not a second orchestrator. It resumes what stalled; it
  never starts a new task, never advances a gate, and never pushes. One exception: the
  release queue's Heartbeat Check acts on this team's OWN ticket exactly as a missed
  `TEAM-QUEUE` would have made her act — including taking the head slot and starting
  STEP 9. It never acts for another team.
```

- [ ] **Step 3: Coordination table rows.** Replace

```
| A task clears Gate 5, is dropped, or is handed over (STEP 13) | `TEAM-DONE` | task name, branch, paths released, and why (done / dropped / handed to <team>) |
```

with

```
| A task clears Gate 5, is dropped, or is handed over (STEP 13) | `TEAM-DONE` | task name, branch, paths released, and why (done / dropped / handed to <team>) |
| Release queue: joining, leaving, a state change to or off `releasing` / `releasing:suite`, abandoning a ticket | `TEAM-QUEUE` (to EVERY Tina) | ticket name, what changed, the current queue order |
| Release queue: the head has been silent for 5 minutes (sent by #2 only) | `TEAM-QUEUE-NUDGE` (direct to the head's session) | ticket name, how long it has been head |
```

- [ ] **Step 4: Insert the Release Queue section** immediately before the line `## Task Classes — Features and the Fast Lane`:

````markdown
## Release Queue — One Release at a Time

**Skip this whole section when you are alone.** No peer Tina in `ListAgents` means no
queue: STEP 9–12 run as written below and no ticket is ever written. A later
`TEAM-HELLO` turns the queue on from the next task that clears Gate 3.

With peers running, teams build and review in parallel, then queue after Gate 3 and
integrate, test and release ONE AT A TIME. The head of the queue runs STEP 9 through
Gate 5 and holds the slot until Gate 5, so deploys never overlap and a rollback only
ever reverts the head's own release. Waiting teams pre-test their branch stacked on
everyone ahead of them, so their turn at the head is usually a release with no suite
run.

### Storage

The queue lives in the git common dir. Every worktree of this clone shares it, and git
never tracks it. Never put it in the working tree.

```bash
Q="$(cd "$(git rev-parse --git-common-dir)" && pwd)/release-queue"
mkdir -p "$Q/tickets" "$Q/abandoned"
```

- A ticket is a directory `tickets/<ns-timestamp>-<team>-<task>/` holding one file,
  `ticket`. Create the directory with `mkdir` (atomic, and it fails if the name exists).
  Get the timestamp with `python3 -c 'import time; print(time.time_ns())'`, because
  macOS `date` has no nanoseconds.
- **Queue order = `ls "$Q/tickets" | sort`. The head is the first entry; your position
  is your line number.** There is no lock file.
- Write a ticket through a temp file so a reader never sees half of it: write
  `ticket.tmp` in the ticket directory, then `mv ticket.tmp ticket`. Refresh `updated`
  on every write.
- Only the owning Tina writes her ticket. The one other change anyone makes is moving a
  stale ticket directory to `abandoned/` (see Stale Tickets).

Ticket file, one `key: value` per line:

| Key | Meaning |
|---|---|
| `team` | this team's name |
| `session` | this session's name as peers see it in `ListAgents` (used for liveness and nudges) |
| `task` | task name |
| `branch` | `feat/<task>` |
| `worktree` | the task worktree path |
| `head` | the branch commit sha this ticket will release |
| `state` | `waiting` · `pretesting` · `ready` · `releasing` · `releasing:suite` |
| `tested_tree` | tree hash of the last PASSing pre-test (empty if none) |
| `spec_on` | the `<ticket-name>@<sha>` list the pre-test was stacked on, in queue order, comma-separated |
| `updated` | ISO timestamp of the last write |
| `nudged` | `<head ticket name> <ISO time>` once this team nudged a silent head (empty otherwise) |

### Lifecycle

1. **Join (STEP 8.5)** — after Gate 3. If `origin/main` moved past the branch's base,
   apply STEP 9's rebase rules first. Then create the ticket (`state: waiting`, `head` =
   `git rev-parse feat/<task>`), broadcast `TEAM-QUEUE`, and send `TEAM-CLAIM` with
   `STEP: queued #n`. Other tasks in the plan keep moving; only this one is parked.
2. **Wait** — pre-test per the Merge Train, then leave the task parked until a
   `TEAM-QUEUE` or a heartbeat tick arrives.
3. **Head** — on every `TEAM-QUEUE` and every heartbeat tick, re-read `tickets/`. If
   your ticket is first and not yet `releasing`: set `state: releasing`, broadcast
   `TEAM-QUEUE`, and run STEP 9 → Gate 5.
4. **Leave** — after Gate 5, a failure, or a drop: `rm -rf "$Q/tickets/<name>"`, then
   broadcast `TEAM-QUEUE` (plus `TEAM-DONE` when the task is finished or dropped).

**A ticket commits to one exact commit.** If the branch changes while waiting (a fix,
more work, a rebase), leave and rejoin at the back. Anyone can check this: `head` vs
`git rev-parse <branch>`. The one exception is the head's own STEP 9 rebase, which is
part of its release: update `head` and carry on. If that rebase changed the dev's own
code, the task goes back to STEP 5, so leave the queue and rejoin after Gate 3.

### Merge Train

**Depth 2.** Only the first two tickets behind the head (positions #2 and #3)
pre-test. Deeper tickets wait, because their stack is the most likely to change.

**Opt-out.** If `/docs/DEVOPS.md` sets `parallel suites: no` (suites that clash over
ports, databases or devices), there is no pre-testing at all. It's a strict queue with
a suite run at the head. Missing setting = `yes`.

**Build the stack** (Tina, git only, no suite):

```bash
git fetch origin
git worktree add --detach .worktrees/pretest-<task> origin/main
cd .worktrees/pretest-<task>
git merge --no-ff --no-edit <head sha of the ticket at #1>   # then #2 … up to yours, in queue order
git merge --no-ff --no-edit feat/<task>
git rev-parse HEAD^{tree}                                      # the stack tree
```

(No `origin` remote → use local `main` as the base.)

- Stack tree equals your `tested_tree` → the earlier pre-test still holds. Update
  `spec_on`, remove the throwaway worktree, and you're done.
- Otherwise set `state: pretesting` and dispatch betty-bugsniff in **pre-test mode**
  with the throwaway worktree path and the stack tree. On PASS (her `tested tree:`
  equals the stack tree), record `tested_tree` and `spec_on` and set `state: ready`. On
  FAIL see Failure Handling. Either way, then
  `git worktree remove --force .worktrees/pretest-<task>`.

**Invalidation.** On every `TEAM-QUEUE` and heartbeat tick, a Tina within depth
compares `spec_on` with the tickets now ahead of her (names and `head` shas).
- Match → nothing to do.
- Mismatch → rebuild the stack and compare its tree with `tested_tree`.
  - Equal → the pre-test still holds; update `spec_on` only. This is the normal case
    when a team ahead released cleanly and left.
  - Different → pre-test again (depth and priority apply).
- A ticket that moves into depth with no `tested_tree` pre-tests.

**Merge conflict while building** → her branch conflicts with a team ahead.
`git merge --abort`, remove the throwaway worktree, and keep her place. Wait for that
team to leave the queue, then dispatch her dev to rebase onto `origin/main`. That
changes her `head`, so she leaves and rejoins at the back.

**Correctness never depends on the pre-test.** At the head, STEP 10 compares the real
merged tree with `tested_tree`, and any difference means betty runs the suite. A stale
pre-test costs a run and nothing else.

### Release Priority

- Pre-tests always run under `nice -n 19`.
- When the head needs a real suite run at STEP 10, it sets `state: releasing:suite` and
  broadcasts `TEAM-QUEUE` BEFORE dispatching betty. Every Tina with a pre-test in flight
  stops her OWN pre-test (`TaskStop` on that betty dispatch, `state: waiting`) and
  restarts it once the head's ticket leaves `releasing:suite`. A team only ever stops
  its own run.
- A head that skipped the suite pre-empts nothing: pushing and watching CI does not
  load this machine.

### Heartbeat Check

Broadcasts are the fast path, not the only one. A `TEAM-QUEUE` can be missed (the
session was mid-turn, or a peer crashed before broadcasting), and a parked task has
nothing else to wake it. So every heartbeat tick with a ticket in the queue re-reads
`tickets/` exactly as if a `TEAM-QUEUE` had arrived:

- **Am I still in line?** Your ticket is in `abandoned/` → delete it and rejoin at the
  back. Its `head` no longer matches `git rev-parse <branch>` → leave and rejoin.
- **Am I the head?** Yes and not yet `releasing` → take the slot and start STEP 9.
- **Is my pre-test current?** Within depth and `spec_on` no longer matches → apply
  Invalidation.
- **Is the head silent?** You are #2, the head is not `releasing`, and its `updated` is
  more than 5 minutes old → nudge (unless `nudged` already names it). `nudged` names it
  and is more than 15 minutes old with no answer → abandon it. The tick measures from
  `updated` and `nudged`, since the broadcast that started the clock may be the one
  that got lost.

With no ticket, the tick only tidies this team's own leftovers. The hourly tick is a
backstop, not the clock.

### Stale Tickets

- **Silent head.** When you become #2, or the head changes while you are #2, and the
  head is not `releasing`: arm a one-shot `CronCreate` (`recurring: false`) about 5
  minutes out. If it fires and that ticket is still head and still not `releasing`,
  send `TEAM-QUEUE-NUDGE` to its `session`, record `nudged` in your ticket, and arm a
  second one-shot 15 minutes out. Any reply from that session or any change to its
  ticket counts as an answer. `CronDelete` pending checks the moment the head changes
  or answers.
- **Abandon.** If the head's session is gone from `ListAgents`, or has not answered 15
  minutes after the nudge: `mv "$Q/tickets/<name>" "$Q/abandoned/"`, broadcast
  `TEAM-QUEUE`, and tell your user which team was skipped and why. If the `mv` fails
  because the ticket is gone, someone else already acted: re-read and move on. Moving
  a ticket only tidies the queue. Never push, deploy or run anything for another team.
- **Coming back.** Finding your own ticket in `abandoned/` → delete it, and rejoin at
  the back if the task still needs releasing.
- **Own leftovers.** At setup and on each heartbeat tick, move this team's tickets
  whose `session` is not this session to `abandoned/`. They are leftovers from a
  crashed earlier run.

### Failure Handling

| Case | Handling |
|---|---|
| Head's Gate 4 suite fails | STEP 11 as usual (reset local `main` to `origin/main`), delete the ticket, broadcast. The fix goes through its gates from STEP 5 and rejoins at the back. |
| Pre-test fails | Re-run pre-test mode on `origin/main` + own branch only. Fails → the branch is at fault: leave, fix from STEP 5, rejoin. Passes → the combination with a team ahead is at fault: keep the place, tell that team which tests broke (information, not an instruction), and wait for it to leave before pre-testing again. |
| Push rejected (`main` moved outside the queue) | Back to STEP 9, keeping the slot. |
| Gate 5 FAIL (pipeline or production) | daisy rolls back first. Then delete the ticket and broadcast; the fix re-enters at STEP 5. |
| Tina is alone | No queue. STEP 9–12 run as written. |

````

- [ ] **Step 5: Fast lane block.** Replace

```
STEP 9-13 unchanged: local ff-only merge, betty's FULL suite on the merged main
          (Gate 4), then daisy's release push and pipeline watch (Gate 5)
```

with

```
STEP 8.5  unchanged: with peers running, the task joins the release queue — FIFO like
          every other task, no jumping (a jump would invalidate every pre-test behind it)
STEP 9-13 unchanged: local ff-only merge, betty's FULL suite on the merged main unless
          a pre-test already passed that exact tree (Gate 4), then daisy's release push
          and pipeline watch (Gate 5)
```

Then replace

```
The two gates that never get skipped are the full suite on the merged `main` and the
release watch — `main` must stay green and the push deploys, and a one-line change is
perfectly capable of breaking both.
```

with

```
The two gates that never get skipped are a full suite on the exact tree being pushed
(at STEP 10, or the pre-test whose tree matches it) and the release watch — `main`
must stay green and the push deploys, and a one-line change is perfectly capable of
breaking both.
```

- [ ] **Step 6: Who Runs Which Tests.** Replace everything from the bullet starting `- **betty-bugsniff at STEP 5 (Gate 2)**` through the end of that section (the paragraph ending `asks the dev for a full run at STEP 3 too.`) with

```
- **betty-bugsniff at STEP 5 (Gate 2)** — decided per task. At planning Tina tags each
  task `full-suite-at-gate2: yes|no`, with a reason when it is `yes` (repo-wide blast
  radius: build config, shared core module, test infrastructure). `yes` → betty runs
  the FULL suite in the worktree, as the authoritative pre-merge run. `no` → Gate 2 is
  betty's test-quality review plus the task's scoped tests and the build, and the
  task's first full run comes later on the integrated tree. Tiny fast-lane tasks carry
  the tag too.
- **betty-bugsniff in pre-test mode** (release queue only) — the FULL suite, niced, on
  a throwaway stack of `origin/main` + the teams ahead + this branch. She reports the
  tested tree hash, and a match at the head makes STEP 10 a no-run.
- **betty-bugsniff at STEP 10** — the FULL suite on the merged local `main`, BEFORE the
  push, unless that exact tree already passed. It runs pre-push because the push
  triggers CI/CD: a red main would already be deploying.

So a task normally gets ONE full run: its pre-test when it queued behind other teams,
or STEP 10 at the head or when alone. A second full run happens only for a task tagged
`full-suite-at-gate2: yes`.
```

- [ ] **Step 7: STEP 5 carries the tag.** Replace

```
STEP 5:  Tina dispatches nick-picker, betty-bugsniff, and sam-shields IN PARALLEL
         (plus polly-pixels when the task's dev was fiona-frontend)
```

with

```
STEP 5:  Tina dispatches nick-picker, betty-bugsniff, and sam-shields IN PARALLEL
         (plus polly-pixels when the task's dev was fiona-frontend). Betty's dispatch
         carries the task's `full-suite-at-gate2` tag: `yes` → full suite in the
         worktree; `no` → test-quality review plus scoped tests and the build
```

- [ ] **Step 8: STEP 8.5 and STEP 9.** Replace

```
         ⛔ GATE 3 — Do NOT proceed until daisy-deployer returns PASS

STEP 9:  Sync and integrate LOCALLY — do not push yet. Tina runs `git fetch origin`.
```

with

```
         ⛔ GATE 3 — Do NOT proceed until daisy-deployer returns PASS

STEP 8.5: RELEASE QUEUE — skipped when alone. Otherwise Tina joins the queue (ticket,
         `TEAM-QUEUE`, claim `STEP: queued #n`) and parks this task. It pre-tests per
         the Merge Train while waiting, and the rest of the plan keeps moving. STEP 9
         starts only when this ticket is the head: `state: releasing`, broadcast, go.

STEP 9:  Sync and integrate LOCALLY — do not push yet. Tina runs `git fetch origin`.
```

In the same STEP 9 paragraph, replace

```
         on the remote. If the rebase changed the dev's own code → re-review from STEP 5;
         if it only resolved trivial conflicts and the scoped tests pass → continue.
```

with

```
         on the remote. If the rebase changed the dev's own code → re-review from STEP 5
         (in a queue: leave it, and rejoin after Gate 3); if it only resolved trivial
         conflicts and the scoped tests pass → continue (in a queue: update the
         ticket's `head` to the rebased sha).
```

- [ ] **Step 9: STEP 10 skip rule.** Replace

```
STEP 10: Tina dispatches betty-bugsniff to run the FULL suite on the local `main` — the
         tree from STEP 9, before it is pushed. May be skipped only when no other commit
         landed on `origin/main` since the branch's last rebase, because then this tree
         is bit-identical to what betty already passed at Gate 2 — otherwise mandatory.
```

with

```
STEP 10: Tina dispatches betty-bugsniff to run the FULL suite on the local `main` — the
         tree from STEP 9, before it is pushed. May be skipped only when that exact tree
         already passed a full run:
         - in a queue: `git rev-parse main^{tree}` equals the ticket's `tested_tree`
         - alone: the task was tagged `full-suite-at-gate2: yes` and no other commit
           landed on `origin/main` since the branch's last rebase
         Otherwise mandatory. In a queue, set `state: releasing:suite` and broadcast
         `TEAM-QUEUE` BEFORE dispatching betty, and set `state: releasing` again (and
         broadcast) when she reports.
```

- [ ] **Step 10: STEP 11, Gate 4, STEP 12, Gate 5, STEP 13.** Replace

```
         Tina dispatches a dev with the failures as highest priority on the task branch →
         the fix passes all gates from STEP 5 before anything is pushed
```

with

```
         Tina dispatches a dev with the failures as highest priority on the task branch →
         the fix passes all gates from STEP 5 before anything is pushed. In a queue:
         delete the ticket and broadcast `TEAM-QUEUE`; the fix rejoins at the back
```

Replace

```
         merged local `main` (or STEP 10 was legitimately skipped per its
         no-other-commits condition)
```

with

```
         merged local `main` (or STEP 10 was legitimately skipped per its
         same-tree condition)
```

Replace

```
STEP 12: RELEASE — Tina dispatches daisy-deployer with the merged commit and the tag (if
         the repo releases by tag). Daisy pushes `main`, WATCHES the CI/CD pipeline to
```

with

```
STEP 12: RELEASE — Tina dispatches daisy-deployer with the merged commit, the tag (if
         the repo releases by tag), and either "no release queue active" or this
         task's ticket name as head of the queue. Daisy pushes `main`, WATCHES the CI/CD pipeline to
```

Replace

```
         verified. If the push is rejected because main moved again → back to STEP 9; if
```

with

```
         verified. If the push is rejected because main moved again → back to STEP 9
         (in a queue, keeping the slot); if
```

Replace

```
         and the production checks. On FAIL: rollback first, then the failure goes to the
         dev and the fix re-enters at STEP 5.
```

with

```
         and the production checks. On FAIL: rollback first, then the failure goes to the
         dev and the fix re-enters at STEP 5 (in a queue: delete the ticket and
         broadcast `TEAM-QUEUE` once the rollback is confirmed).
```

Replace

```
STEP 13: After Gate 5, Tina removes the worktree, deletes the branch (local and remote),
         and refreshes the main checkout (`git fetch origin && git pull --ff-only`).
```

with

```
STEP 13: After Gate 5, Tina removes the worktree, deletes the branch (local and remote),
         and refreshes the main checkout (`git fetch origin && git pull --ff-only`). In
         a queue: delete the ticket, broadcast `TEAM-QUEUE`, then `TEAM-DONE`.
```

- [ ] **Step 11: Verify.**

Run:
```bash
f=skills/tdd-agent-team/references/workflow.md
grep -c 'TEAM-QUEUE' $f
grep -n '^## Release Queue\|^### Heartbeat Check\|^STEP 8.5\|full-suite-at-gate2\|parallel suites' $f
grep -c 'time_ns\|ticket.tmp\|recurring: false\|nudged\|nice -n 19' $f
grep -n '%N' $f
grep -n 'no-other-commits\|asks the dev for a full run at STEP 3' $f
```
Expected: first count ≥ 15; second shows the section, subsection, STEP 8.5 and several tag/setting lines; third count ≥ 8; the `%N` grep prints only the "macOS `date` has no nanoseconds" context line or nothing (no `date +%s%N` command); the last grep prints nothing.

- [ ] **Step 12: Commit**

```bash
git add skills/tdd-agent-team/references/workflow.md
git commit -m "feat(workflow): release queue — one release at a time, merge-train pre-tests"
```

---

### Task 2: SKILL.md points at the queue

**Files:**
- Modify: `skills/tdd-agent-team/SKILL.md` (Setup 5, 7, 8; roster rows betty and daisy; rules at ~L55-58)

**Interfaces:**
- Consumes: section and message names from Task 1.

- [ ] **Step 1: Planning tags the task.** Replace

```
5. Break the task into small units with BDD scenarios (Given/When/Then)
```

with

```
5. Break the task into small units with BDD scenarios (Given/When/Then), and tag each one `full-suite-at-gate2: yes|no` — `yes` only for a repo-wide blast radius (build config, shared core module, test infrastructure), with the reason
```

- [ ] **Step 2: Heartbeat prompt.** In Setup 7, replace

```
whose prompt re-reads the task list and re-dispatches any in_progress task whose agent died (rate limit, timeout, error).
```

with

```
whose prompt re-reads the task list and re-dispatches any in_progress task whose agent died (rate limit, timeout, error), then runs the release queue's Heartbeat Check (re-read the queue as if a `TEAM-QUEUE` arrived, act on this team's own ticket, tidy its own leftover tickets).
```

- [ ] **Step 3: Coordination events.** In Setup 8, replace

```
`TEAM-CLAIM` when a task starts, `TEAM-DONE` when it ends, and takeover requests negotiated per `references/workflow.md`.
```

with

```
`TEAM-CLAIM` when a task starts, `TEAM-DONE` when it ends, `TEAM-QUEUE` to every Tina on each release-queue change, and takeover requests negotiated per `references/workflow.md`.
```

- [ ] **Step 4: Roster rows.** Replace the betty row

```
| betty-bugsniff (QA) | tdd-agent-team:betty-bugsniff | BDD scenarios, branch name, worktree path |
```

with

```
| betty-bugsniff (QA) | tdd-agent-team:betty-bugsniff | Gate 2: BDD scenarios, branch name, worktree path, the task's `full-suite-at-gate2` tag (yes or no) — pre-test mode: "pre-test", the throwaway worktree path, and the expected stack tree hash |
```

and in the daisy row replace

```
release (STEP 12): the merged commit sha, tag if any, and that Gate 4 passed |
```

with

```
release (STEP 12): the merged commit sha, tag if any, that Gate 4 passed, and either "no release queue active" or this task's ticket name as head of the queue |
```

- [ ] **Step 5: Integration bullet.** Replace

```
Full sequence in `references/workflow.md`.
- Coordination is event-driven:
```

with

```
Full sequence in `references/workflow.md`.
- With a peer Tina running, releases go through the release queue (`references/workflow.md`, Release Queue): a task joins after Gate 3, only the head of the queue integrates, tests and pushes, and it holds the slot through Gate 5. Waiting teams pre-test on the stack ahead of them. FIFO for every task, tiny ones included. Alone → no queue.
- Coordination is event-driven:
```

- [ ] **Step 6: Heartbeat and test-allocation bullets.** Replace

```
- Keep the heartbeat honest: it only resumes stalled dispatches — it never starts a new task, advances a gate, or pushes. After a rate limit, resume one agent at a time rather than re-firing the whole parallel fan-out.
- Test runs are allocated, not repeated: devs run their task's scope plus the build, betty-bugsniff runs the full suite in the worktree at Gate 2 and again on the merged local `main` at STEP 10, before the push. Ask a dev for a full run only when the task's blast radius is genuinely repo-wide, and say so in the task details.
```

with

```
- Keep the heartbeat honest: it only resumes stalled dispatches — it never starts a new task, advances a gate, or pushes. The one exception is the release queue's Heartbeat Check, which acts on this team's OWN ticket as a missed `TEAM-QUEUE` would have (up to taking the head slot) and never for another team. After a rate limit, resume one agent at a time rather than re-firing the whole parallel fan-out.
- Test runs are allocated, not repeated: devs run their task's scope plus the build. betty-bugsniff runs the full suite once per task on the integrated tree — its pre-test in the queue, or STEP 10 on the merged local `main` before the push — and skips STEP 10 when that exact tree already passed. Her Gate 2 run is full only for tasks tagged `full-suite-at-gate2: yes`.
```

- [ ] **Step 7: Verify.**

Run: `grep -c 'TEAM-QUEUE\|release queue\|Release Queue\|full-suite-at-gate2' skills/tdd-agent-team/SKILL.md && grep -n 'full suite in the worktree at Gate 2 and again' skills/tdd-agent-team/SKILL.md`
Expected: count ≥ 8; second grep prints nothing.

- [ ] **Step 8: Commit**

```bash
git add skills/tdd-agent-team/SKILL.md
git commit -m "feat(skill): route releases through the release queue"
```

---

### Task 3: betty — scoped Gate 2 and pre-test mode

**Files:**
- Modify: `agents/betty-bugsniff.md`

**Interfaces:**
- Consumes: the dispatch contents from Task 2's roster row.
- Produces: `PASS`/`FAIL` first line, plus `tested tree: <hash>` for any full run (read by Tina at the Merge Train and STEP 10).

- [ ] **Step 1: Frontmatter description.** Replace

```
description: QA on the TDD agent team — validates test quality and coverage, runs the suite in the task worktree, and runs integration tests on main after merges.
```

with

```
description: QA on the TDD agent team — validates test quality and coverage, runs the suite in the task worktree, pre-tests release-queue stacks, and runs integration tests on main after merges.
```

- [ ] **Step 2: Gate 2 scope.** Replace

```
3. Run the FULL test suite in the worktree and verify everything passes. This is the
   first and only full run before the merge — the dev ran only the tests in the task's
   scope — so a failure outside that scope is a real regression to report, not noise
```

with

```
3. If the dispatch says `full-suite-at-gate2: yes` (or says nothing): run the FULL test
   suite in the worktree and verify everything passes. The dev ran only the tests in
   the task's scope, so a failure outside that scope is a real regression to report,
   not noise. If it says `no`: run only the task's scoped tests (the new tests plus
   the existing tests for the modules touched) and the build. The task's full run
   happens later on the integrated tree
```

- [ ] **Step 3: Pre-test mode section.** Insert before `## Integration Testing (merged main, before it is pushed)`:

```markdown
## Pre-Test Mode (release queue)

When the dispatch says "pre-test", Tina gives you a throwaway worktree (detached HEAD:
`origin/main` with the branches of the teams ahead in the release queue merged in, then
this task's branch) and the stack tree hash she expects.

1. In that worktree, confirm `git rev-parse HEAD^{tree}` equals the expected hash. A
   mismatch → FAIL without running anything
2. Run the FULL suite under `nice -n 19`, in the FOREGROUND — never backgrounded or
   detached. Another team's release has priority, and Tina may stop you to make room;
   a foreground run stops with you
3. Skip the review checklist (Gate 2 covered it). Do not edit, commit, or push anything
4. Report per Final Response, with the `tested tree:` line

```

- [ ] **Step 4: Final response line.** Replace

```
- **PASS**: one-line confirmation for the branch (or for main, when integration testing)
```

with

```
- **PASS**: one-line confirmation for the branch (or for main, when integration testing)
- After any FULL run (Gate 2 full, pre-test, main), add a line `tested tree: <hash>` with `git rev-parse HEAD^{tree}` of the tree you ran on
```

- [ ] **Step 5: Verify.**

Run: `grep -n 'Pre-Test Mode\|nice -n 19\|FOREGROUND\|tested tree:\|full-suite-at-gate2' agents/betty-bugsniff.md && head -4 agents/betty-bugsniff.md`
Expected: all five patterns present; frontmatter still has `name: betty-bugsniff` and the description still ends with `not for general use.`

- [ ] **Step 6: Commit**

```bash
git add agents/betty-bugsniff.md
git commit -m "feat(betty): pre-test mode and per-task Gate 2 scope"
```

---

### Task 4: daisy — confirm head of queue before pushing

**Files:**
- Modify: `agents/daisy-deployer.md`

- [ ] **Step 1: Release step 2.** Replace

```
   merge is not a fast-forward ahead of `origin/main`, STOP and report FAIL without
   pushing
```

with

```
   merge is not a fast-forward ahead of `origin/main`, STOP and report FAIL without
   pushing. The dispatch must also say either "no release queue active" or name this
   task's ticket as head of the queue. When it names a ticket, check that it is the
   first entry of `ls "$(git rev-parse --git-common-dir)/release-queue/tickets" | sort`.
   If the dispatch says neither, or the ticket is not first, STOP and report FAIL
   without pushing
```

- [ ] **Step 2: Verify.**

Run: `grep -n 'release-queue/tickets\|no release queue active' agents/daisy-deployer.md`
Expected: both lines present in the release section.

- [ ] **Step 3: Commit**

```bash
git add agents/daisy-deployer.md
git commit -m "feat(daisy): push only as head of the release queue"
```

---

### Task 5: DEVOPS template setting and README

**Files:**
- Modify: `skills/tdd-agent-team/references/templates.md` (before `### Release / Production (CI/CD-triggered)`)
- Modify: `README.md` (paragraph at L22; new paragraph after L26)

- [ ] **Step 1: Template block.** Insert before the line `### Release / Production (CI/CD-triggered)`:

````markdown
### Test Suite (release queue)

Optional. It matters only when several Tinas share this repo on one machine. Missing
means `yes`.

```markdown
### Test Suite
- **parallel suites**: yes   <!-- no = full-suite runs clash (fixed ports, shared database, one device): no pre-testing, the release queue runs one suite at a time at the head -->
```

````

- [ ] **Step 2: README test-run sentence.** Replace

```
Devs run only their task's scope plus the build; the full suite runs twice, both times by QA — in the worktree at the review gate, and on the locally merged `main` just before it is pushed.
```

with

```
Devs run only their task's scope plus the build; QA runs the full suite on the integrated tree just before it is pushed, and in the worktree at the review gate only for tasks whose blast radius is repo-wide.
```

- [ ] **Step 3: README queue paragraph.** Insert after the paragraph that starts `That makes it worth running teams with different focuses`:

```
Teams on the same machine release through a queue instead of racing each other to the push. A task joins after its deploy check, and only the team at the head integrates, tests, pushes and watches production — one release at a time, so a rollback only ever reverts its own release. While they wait, the next two teams in line pre-test their branch stacked on everything ahead of them. When their turn comes the merged tree usually matches what already passed, and they push without running the suite again. The queue is a handful of files inside the clone's `.git` directory, every change is broadcast to every Tina, and each Tina's hourly heartbeat also checks her place in line in case a message got lost. A Tina running alone never sees any of this.
```

- [ ] **Step 4: Verify.**

Run: `grep -n 'parallel suites' skills/tdd-agent-team/references/templates.md && grep -n 'release through a queue\|full suite runs twice' README.md`
Expected: template line present; README shows the new paragraph and no "full suite runs twice".

- [ ] **Step 5: Commit**

```bash
git add skills/tdd-agent-team/references/templates.md README.md
git commit -m "docs: release queue in README and DEVOPS template"
```

---

### Task 6: Cross-file consistency check

- [ ] **Step 1: Names match everywhere.**

Run:
```bash
grep -rhoE 'TEAM-QUEUE(-NUDGE)?|full-suite-at-gate2|parallel suites|releasing:suite|tested tree:|Heartbeat Check|STEP 8\.5' skills agents README.md | sort | uniq -c
grep -rn 'TEAM_QUEUE\|full_suite_at_gate2\|full-suite-at-gate-2\|parallel_suites\|releasing-suite' skills agents README.md
```
Expected: first command lists each canonical name. The second prints nothing (no misspelt variants).

- [ ] **Step 2: Spec coverage walk.** Open the spec's "Files Touched" table and check that each row's change is visible in the corresponding diff (`git diff HEAD~5 -- <file>`). Fix inline and amend the relevant commit only if something is missing.
