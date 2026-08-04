# Dev Role Split & New Roles — Design

Date: 2026-08-04
Status: approved design, pre-implementation
Scope: part of the `tdd-agent-team` plugin

## Problem

One generic dev role (billy-builder) handles every task. Domain-specific work
(frontend/webdesign, backend, firmware, mobile) benefits from specialized system
prompts, and two team gaps exist: nobody looks at rendered UI (betty runs test suites,
not browsers) and docs updates ride along unsystematically with dev tasks.

## Decisions (from brainstorming)

1. Split the dev role into five: generic (billy-builder, unchanged, the fallback) plus
   frontend, backend, firmware, mobile — each adapted from billy-builder's body.
2. Add a design reviewer (visual QA, frontend tasks only) and a docs writer.
3. **Fiona designs while building**: no separate up-front designer role. The frontend
   dev makes visual decisions during implementation, constrained by the existing design
   system; the design reviewer catches misses.
4. Docs writer runs **after Gate 2, pre-merge** — docs land in the same branch as the
   code they describe; docs-only commits, no re-review loop.

## Roster

| Agent | Role | File |
|---|---|---|
| billy-builder | Generic dev — unchanged fallback | `agents/billy-builder.md` (existing) |
| fiona-frontend | Frontend dev, webdesign focus | `agents/fiona-frontend.md` |
| benny-backend | Backend dev | `agents/benny-backend.md` |
| frank-firmware | Firmware/embedded dev | `agents/frank-firmware.md` |
| mandy-mobile | Mobile dev | `agents/mandy-mobile.md` |
| polly-pixels | Design reviewer (visual QA) | `agents/polly-pixels.md` |
| wally-wordsmith | Docs writer | `agents/wally-wordsmith.md` |

Names follow the existing alliteration convention; no first-letter collisions with
existing teammates.

## Role file content

### Dev specializations (fiona, benny, frank, mandy)

Self-contained copies of billy-builder's body (agent prompts cannot import shared
text): same TDD workflow (RED/GREEN/REFACTOR), design principles, code-quality rules,
worktree rules, and DONE/BLOCKED final-response contract. Each adds one **Domain
focus** section:

- **fiona-frontend**: semantic HTML; responsive layout; spacing/typography consistency
  with the existing design system; accessibility (keyboard, contrast, ARIA); reuse
  existing components before writing new CSS; implement loading/empty/error states;
  verify visually in a browser when browser tooling is available — tests alone don't
  prove visual correctness. She makes the visual design decisions herself.
- **benny-backend**: API contracts and versioning; input validation at boundaries;
  reversible migrations; meaningful error responses that don't leak internals;
  integration tests against real interfaces over mocks.
- **frank-firmware**: resource constraints (static allocation, stack awareness);
  hardware abstraction so logic tests off-target; interrupt/timing safety; defensive
  register access; host-side tests against the HAL when no target is attached.
- **mandy-mobile**: platform conventions (iOS/Android/cross-platform); lifecycle and
  state restoration; offline and poor-network behavior; battery and permissions
  discipline; tests via the platform's test runner/simulator.

### polly-pixels (design reviewer)

Reviewer pattern (PASS/FAIL contract, findings addressed to the dev, relayed verbatim
by Tina). Reviews the **rendered UI**, not the diff: builds/serves the app from the
task worktree and inspects it with browser tooling when available; otherwise reviews
markup/styles/screenshots and states the limitation in the verdict. Checks: layout and
spacing, responsive breakpoints, visual consistency with the rest of the app,
accessibility basics, presence of loading/empty/error states.

### wally-wordsmith (docs writer)

Works in the task worktree after all reviews pass. Updates READMEs and docs affected
by the change; commits are docs-only; never touches source code. DONE/BLOCKED
contract. On DONE reports which docs changed; reporting "no docs affected" is a valid
DONE.

## Tina integration

- Dispatch table gains the new rows (subagent types `tdd-agent-team:<name>`, bare
  names as fallback, same as existing teammates).
- Dev selection rule: **pick the dev whose domain matches the task; use billy-builder
  when no domain fits or a task genuinely spans domains** — but prefer splitting
  mixed-domain tasks by domain at planning time (domain splits usually have zero file
  overlap, so they parallelize).
- Gate 2: frontend tasks get polly-pixels as a 4th parallel reviewer. Gate 2 = ALL
  dispatched reviewers PASS (3 normally, 4 for frontend tasks).
- workflow.md: new STEP 6.5 — after Gate 2, dispatch wally-wordsmith in the task
  worktree; docs-only commits, no re-review loop; then continue to the deploy gate.
- README: roster table updated with all new roles.

## Out of scope (YAGNI)

- No dedicated up-front designer role (decision 3).
- No new memory, workflow-gate, or dreaming changes beyond STEP 6.5 and the Gate 2
  reviewer count.
- billy-builder's content is not modified.

## Verification (manual)

1. Frontmatter of every new agent file parses (name/description present).
2. Tina's dispatch table and workflow.md reference only agent names that exist in
   `agents/`.
3. After syncing `agents/*.md` to `~/.claude/agents/`, the harness lists the new agent
   types by name.
4. Dry-run check: ask Tina (in a scratch session) to plan a task mixing a UI tweak and
   an API change — she should split it into a fiona task and a benny task, and list
   polly as a 4th reviewer only for the fiona task.
