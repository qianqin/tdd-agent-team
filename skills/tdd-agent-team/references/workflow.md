# Task Workflow Reference

## Branch & Worktree Rules

- The main checkout stays on `main` at all times — Tina never checks out feature branches there
- Before creating the first worktree, ensure `.worktrees/` is in `.gitignore` (add it if missing)
- Each task gets its own worktree: `git worktree add .worktrees/<task> -b feat/<task>`
- The dev, QA test runs, and deploy builds all happen inside the task's worktree
- Code review and security review read the diff (`git diff main...feat/<task>`) — no checkout needed
- After merge: `git worktree remove .worktrees/<task>` and delete the branch

## Workflow per Task (NO SKIPPING STEPS)

```
STEP 1:  Tina creates the task worktree: git worktree add .worktrees/<task> -b feat/<task>
STEP 2:  Tina dispatches a dev with BDD scenarios, worktree path, and branch name

STEP 3:  Dev writes tests first (RED), implements (GREEN), refactors (REFACTOR), commits
STEP 4:  Dev's final response reports DONE with test and build results

         ⛔ GATE 1 — Do NOT proceed until the dev reports DONE (a BLOCKED response
         means Tina resolves the blocker and re-dispatches)

STEP 5:  Tina dispatches nick-picker, betty-bugsniff, and sam-shields IN PARALLEL
         (plus polly-pixels when the task's dev was fiona-frontend)
STEP 6:  All dispatched reviewers must return PASS. Any FAIL → Tina sends the findings to the dev
         (same agent if the harness allows, else a fresh dev) → dev fixes → re-review from STEP 5

         ⛔ GATE 2 — Do NOT proceed until ALL dispatched reviewers return PASS
         (3 normally, 4 when polly-pixels was dispatched for a frontend task)

STEP 6.5: Tina dispatches wally-wordsmith in the task worktree to update any docs
         affected by the change. Docs-only commits; no re-review loop. "No docs
         affected" is a valid DONE.

STEP 7:  Tina dispatches daisy-deployer with the worktree path for deploy & verification
STEP 8:  If deploy test fails → daisy rolls back, reports failure details → Tina sends
         them to the dev → fix → re-review from STEP 5

         ⛔ GATE 3 — Do NOT proceed until daisy-deployer returns PASS

STEP 9:  Sync first: if main has advanced since the branch was cut, Tina dispatches the
         task's dev to merge main into the branch inside the worktree, resolve any
         conflicts, and re-run the full suite. If the merge changed the dev's own code
         → re-review from STEP 5; if it only resolved trivial conflicts and tests pass
         → continue. Then Tina merges the branch to main (now conflict-free), removes
         the worktree, deletes the branch.
STEP 10: Tina dispatches betty-bugsniff to run the full integration test suite on main.
         May be skipped when the merge was a fast-forward and no other merge landed
         since the branch was cut — otherwise mandatory.
STEP 11: If integration tests fail → STOP all new task assignments → Tina dispatches a dev
         with the failures as highest priority → fix must pass all gates before resuming

         ⛔ GATE 4 — Do NOT proceed until betty-bugsniff reports integration tests PASS
         on main (or STEP 10 was legitimately skipped per its fast-forward condition)

STEP 12: Tina assigns the next task
```

After each step, state: "✅ [task-name] STEP N complete. Proceeding to STEP N+1."

## Progress Tracking

Track every task in the integrated task list (TaskCreate/TaskUpdate): one entry per
task, named `<task> — STEP N`, updated as each gate clears. In_progress when the dev
is dispatched, completed only after Gate 4. Check the task list before dispatching
new work — never rely on transcript memory for gate state.

## Task Breakdown Guidelines

1. Parallel tasks must have ZERO file overlap — parallel devs never work on the same files; tasks that share files must run sequentially
2. Dependencies between tasks must be explicit
3. Every task needs BDD scenarios written by Tina BEFORE assignment

## BDD Scenario Format

```gherkin
Feature: <feature name>

  Scenario: <scenario description>
    Given <precondition>
    When <action>
    Then <expected outcome>
```

Write multiple scenarios per task covering happy path, error cases, and edge cases.
