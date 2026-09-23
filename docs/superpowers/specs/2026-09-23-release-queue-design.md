# Release Queue — Design

Date: 2026-09-23
Status: approved design, pre-implementation
Scope: part of the `tdd-agent-team` plugin

## Problem

Concurrent Tinas on one machine release optimistically: each runs STEP 9–12
(fetch → ff-merge → full suite → push → pipeline watch) on its own. The first push
wins; the loser is rejected, rebases, and pays for another full suite — and may push
while the winner's pipeline is still deploying, so a rollback can revert the other
team's release. The full suite is the bottleneck, and the same code gets tested more
than once.

## Goal

Teams build and review in their worktrees in parallel, then **queue** for release and
integrate, test, and release one at a time. Minimise the number of full-suite runs on
identical code.

## Decisions (from brainstorming)

1. **Same machine only.** All competing sessions are worktrees of one clone, sharing
   one git common dir. The queue is files there — no server, no remote refs.
2. **Join after Gate 3.** A task enters the queue once all worktree work (dev, reviews,
   docs, deploy check) is done. The team at the head runs integration, test and
   release (STEP 9 → Gate 5).
3. **The head holds the slot through Gate 5.** Deploys never overlap; a rollback only
   ever reverts the head's own release.
4. **Merge train.** Waiting teams pre-test their branch stacked on everyone ahead, so
   their turn is usually a no-run release.
5. **No shared pass cache.** The only reuse that pays is a team's own pre-test; the
   tested tree hash lives in the ticket, and the ticket is deleted after release.
6. **Gate 2's full run is a per-task choice** — kept only for repo-wide blast radius.
7. **FIFO for everyone, including tiny fast-lane tasks.** Jumping would invalidate
   every pre-test behind the jumper.
8. **Release has priority over pre-tests** for CPU and memory, always.
9. **Every queue change is broadcast to every Tina.** The queue directory is the source
   of truth; messages are only the wake-up.
10. **Skipped entirely when alone.** No peer Tina → no queue, same as the existing
    coordination protocol.

## Storage

```
$(git rev-parse --git-common-dir)/release-queue/
  tickets/<ns-timestamp>-<team>-<task>/ticket
  abandoned/<same name>/ticket
```

- `git rev-parse --git-common-dir` resolves to `<main clone>/.git` from every worktree.
  Git never tracks anything under `.git/`, so the queue cannot be committed and needs no
  `.gitignore` entry. The queue MUST NOT be placed anywhere in the working tree.
- A ticket directory is created with `mkdir` (atomic). The `<ns-timestamp>` prefix makes
  names unique and gives the order: **queue order = sorted ticket names; the head is
  the first ticket.** There is no separate lock.
- Create `release-queue/tickets` and `release-queue/abandoned` with `mkdir -p` on first
  use.

**Ticket file** (`key: value` lines):

| Key | Meaning |
|---|---|
| `team` | Tina's team name |
| `session` | the session name as `ListAgents` shows it (used for liveness and nudges) |
| `task` | task name |
| `branch` | `feat/<task>` |
| `worktree` | worktree path |
| `head` | branch commit sha this ticket will release |
| `state` | `waiting` · `pretesting` · `ready` · `releasing` · `releasing:suite` |
| `tested_tree` | tree hash of the last PASSing pre-test (empty if none) |
| `spec_on` | the `<ticket-name>@<sha>` list the pre-test was stacked on, in order |
| `updated` | ISO timestamp of the last write |

Only the owning Tina writes her ticket. The one exception is moving a stale ticket to
`abandoned/` (see Stale and Blocking Tickets).

## Lifecycle

1. **Join** — after Gate 3, with the branch rebased on `origin/main`. Tina creates the
   ticket (`state: waiting`, `head` = branch sha), broadcasts `TEAM-QUEUE`, and updates
   her claim to `STEP: queued #n`. Other tasks in her plan keep moving; only this one
   is parked.
2. **Wait** — pre-test per the Merge Train section, then idle on this task until a
   broadcast arrives.
3. **Head** — on any broadcast, each Tina re-reads `tickets/`. The first ticket's
   owner sets `state: releasing`, broadcasts, and runs:
   - STEP 9: `git fetch origin`, dev rebases onto `origin/main` if needed, ff-merge
     into local `main`.
   - STEP 10: compare `git rev-parse main^{tree}` to `tested_tree`. Equal → skip the
     suite (Gate 4 passes on the pre-test). Different → set `state: releasing:suite`,
     broadcast, and betty runs the full suite.
   - STEP 12: daisy pushes and watches the pipeline through to Gate 5.
4. **Leave** — after Gate 5, a failure, or a drop: delete the ticket directory, then
   broadcast `TEAM-QUEUE` (plus `TEAM-DONE` when the task is finished or dropped).

**The ticket commits to one exact commit.** If the branch changes after joining — a
fix, more work, a rebase — the ticket is invalid: leave the queue and rejoin at the
back. Anyone can check this by comparing `head` with `git rev-parse <branch>`.

## Merge Train

**Build the pre-test tree.** In a throwaway worktree: start from `origin/main`, then
`git merge --no-ff` the `head` of each ticket ahead, in queue order, then this branch.
Betty runs the full suite there. On PASS, record `tested_tree` (the tree hash) and
`spec_on`, and set `state: ready`. Remove the throwaway worktree afterwards.

**Train depth 2.** Only the first two tickets behind the head pre-test. Deeper tickets
wait. Their stack is the most likely to change.

**Invalidation.** On every broadcast, a waiting Tina compares `spec_on` with the
current tickets ahead of her (names and `head` shas). Match → nothing to do. Mismatch
→ rebuild and pre-test again (subject to depth and priority).

**Merge conflict while building** → her branch conflicts with a team ahead. She keeps
her place, waits for that team's release, then has her dev rebase. That changes her
`head`, so she leaves and rejoins at the back.

**Correctness never depends on the pre-test.** At the head, the rebased tree normally
has the same hash as `tested_tree`, because the teams ahead pushed exactly their
recorded `head`s. If it doesn't (rebase and merge resolved differently, or someone
outside the queue pushed), betty runs the suite. A stale pre-test costs a run and
nothing else.

**Opt-out.** `/docs/DEVOPS.md` may set `parallel suites: no` — for suites that clash
over ports, databases or fixed resources. Then there is no pre-testing, only the strict
queue with a run at the head.

## Release Priority

- Pre-tests always run under `nice -n 19`.
- When the head needs a real suite run, it sets `state: releasing:suite` and
  broadcasts **before** starting. Every Tina with a pre-test in flight stops her OWN
  pre-test, then restarts it when the head's ticket leaves `releasing:suite`. A team
  only ever stops its own run.
- A head that skipped the suite (tree hash matched) pre-empts nothing: pushing and
  watching CI does not load this machine.

## Broadcasts

A new event joins the coordination table in `references/workflow.md`:

| When | Message | Contents |
|---|---|---|
| Joining, leaving, a state change to `releasing` / `releasing:suite` / off it, abandoning a ticket | `TEAM-QUEUE` | ticket name, what changed, the current queue order |

It is sent to **every** Tina from `ListAgents`, not only to the next in line: a
departure mid-queue invalidates the pre-tests of everyone behind it. Receivers treat it
as a wake-up and re-read `tickets/`; the message itself authorises nothing.

## Stale and Blocking Tickets

- **Silent head.** When a broadcast makes a ticket the head and its state is not
  `releasing` within **5 minutes** of that broadcast, the #2 owner sends a direct `TEAM-QUEUE-NUDGE`
  to the head's `session`.
- **Abandon.** If that session is gone from `ListAgents`, or does not answer within
  **15 minutes** of the nudge, the #2 owner moves the ticket to `abandoned/`, broadcasts
  `TEAM-QUEUE`, and tells her user which team was skipped and why. Moving a ticket
  only tidies the queue: no one pushes, deploys or runs anything for another team, so
  the existing "never act for a peer" rule holds.
- **Coming back.** A Tina who finds her ticket in `abandoned/` deletes it and rejoins at
  the back.
- **Own leftovers.** At setup, and on each heartbeat tick, Tina moves her own team's
  tickets whose `session` is not her current one to `abandoned/` — leftovers from a
  crashed earlier run.

## Failure Handling

| Case | Handling |
|---|---|
| Head's Gate 4 suite fails | Reset local `main` to `origin/main` (as today), delete the ticket, broadcast. The fix goes through its gates from STEP 5 and rejoins at the back. Pre-tests stacked on it are rebuilt. |
| Pre-test fails | Re-run on `origin/main` + own branch only. Fails → the branch is at fault: leave, fix, rejoin. Passes → the combination with a team ahead is at fault: keep the place, tell that team which tests broke (as information, not an instruction), and wait for its release before pre-testing again. |
| Push rejected (`main` moved outside the queue) | Back to STEP 9 while keeping the slot. |
| Pipeline or production fails (Gate 5 FAIL) | Daisy rolls back first. Then delete the ticket and broadcast; the fix re-enters at STEP 5. |
| Tina is alone (no peer in `ListAgents`) | No queue: STEP 9–12 run as today. A later `TEAM-HELLO` turns the queue on from the next task that finishes Gate 3. |

## Gate 2 Per-Task Choice

At task creation Tina tags each task `full-suite-at-gate2: yes|no` and gives a reason
when it is `yes` (repo-wide blast radius: build config, shared core module, test
infrastructure).

- `yes` → betty runs the full suite in the worktree at Gate 2, as today.
- `no` → Gate 2 is betty's test-quality review plus the task's scoped tests and build.
  The train run is the task's only full run.

This applies to tiny fast-lane tasks too. They keep their slimmed reviewer set and
queue FIFO like everyone else.

## Files Touched

| File | Change |
|---|---|
| `skills/tdd-agent-team/references/workflow.md` | New **Release Queue** section (everything above). STEP 8→9 gains "join queue, wait for head". STEP 10's skip rule becomes the `tested_tree` comparison. STEP 12's rejection rule keeps the slot. Gate 2 uses the per-task tag. `TEAM-QUEUE` / `TEAM-QUEUE-NUDGE` join the event table. The fast-lane block notes FIFO queueing. |
| `skills/tdd-agent-team/SKILL.md` | The integration and coordination bullets point at the queue. The heartbeat also tidies the team's own stale tickets and never advances the queue. The betty row in the roster table covers the pre-test mode. |
| `agents/betty-bugsniff.md` | New pre-test mode: run niced in the given throwaway worktree, stoppable on request, report PASS/FAIL with the tested tree hash. The Gate 2 run is scoped when the task says `full-suite-at-gate2: no`. |
| `agents/daisy-deployer.md` | Before pushing, confirm that the dispatch says this team holds the head of the queue (or that no queue is active). |
| `skills/tdd-agent-team/references/templates.md` (DEVOPS.md template) | `parallel suites: yes|no` setting, default `yes`. |
| `README.md` | A short paragraph on the release queue. |

## Out of Scope

- Sessions on different machines or cloud sandboxes (would need a remote queue, e.g.
  git refs on `origin`).
- Batching several teams' branches into one release.
- Queue priorities or jumping the queue.
- A shared cache of passed trees across teams.
