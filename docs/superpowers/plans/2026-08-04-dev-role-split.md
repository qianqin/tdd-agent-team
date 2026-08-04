# Dev Role Split & New Roles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split the dev role into five domain specializations and add a design reviewer and docs writer, with Tina dispatching the right agent per task.

**Architecture:** Four new dev agents share billy-builder's exact body plus a per-role intro and a `## Domain Focus` section (agent prompts are self-contained; the shared body is generated once from a script so the copies can't drift at creation time). Two new non-dev agents follow the existing reviewer/worker patterns. Tina's SKILL.md gains a dev-selection rule and reviewer changes; workflow.md gains STEP 6.5 (docs) and a Gate 2 reviewer-count change.

**Tech Stack:** Claude Code plugin markdown (agents + skills), bash heredoc generation.

**Spec:** `docs/superpowers/specs/2026-08-04-dev-role-split-design.md` — read it before starting.

## Global Constraints

- billy-builder's file content is NOT modified.
- Agent names exactly: `fiona-frontend`, `benny-backend`, `frank-firmware`, `mandy-mobile`, `polly-pixels`, `wally-wordsmith`.
- All agents: `agents/<name>.md`, frontmatter `name` + `description`, description ends with "not for general use." like existing teammates.
- Dev DONE/BLOCKED and reviewer PASS/FAIL final-response contracts stay word-compatible with existing agents (Tina parses first lines).
- Subagent types referenced as `tdd-agent-team:<name>` with bare `<name>` fallback, matching existing convention.

---

### Task 1: Four dev specialization agents

**Files:**
- Create: `agents/fiona-frontend.md`, `agents/benny-backend.md`, `agents/frank-firmware.md`, `agents/mandy-mobile.md`

**Interfaces:**
- Produces: the four dev agent types Task 4's dispatch table references. Final response contract: first line `DONE` or `BLOCKED` (identical to billy-builder).

- [ ] **Step 1: Generate the four files from one shared body**

Run this script from the repo root. It writes the shared body (billy-builder's body verbatim, with a role-specific intro sentence and an inserted `## Domain Focus` section) for each role:

```bash
set -euo pipefail

write_dev() {
  local file="$1" name="$2" desc="$3" intro="$4" focus="$5"
  cat > "agents/${file}" <<EOF
---
name: ${name}
description: ${desc}
---

${intro}

## Workflow

1. **Read** the BDD scenarios in your task
2. **RED** — Write failing tests that match the BDD scenarios
3. **GREEN** — Write the minimum code to make tests pass
4. **REFACTOR** — Clean up without changing behavior
5. **Verify** — Run the full build and test suite. Fix anything that fails.
6. **Commit** — Small, descriptive messages: \`feat(scope): what changed\`
7. **Report** — see Final Response below

## Design Principles

1. **Don't overengineer** — simple beats complex
2. **One correct path** — no fallback chains, low cyclomatic complexity
3. **Clarity over compatibility** — clear code beats clever solutions
4. **Throw errors** — fail fast when preconditions aren't met
5. **Separation of concerns** — single responsibility per function/module/file
6. **Surgical changes** — minimal, focused fixes
7. **Fix root causes** — address underlying issues, not symptoms

## Domain Focus

${focus}

## Code Quality

- Comments explain WHY, not HOW — no comments about previous versions
- Check for ripple effects: assumptions, usage, tests, build tooling, READMEs, CI
- Complete the entire task — don't leave items for later
- Some duplication is OK; refactor when the same logic appears in 3+ places

## Rules

- Work ONLY inside your assigned worktree, on your assigned branch — never touch the main checkout or files outside your assigned scope
- If \`/docs/OSS.md\` exists in the project, read it before adding any new dependencies
- If the BDD scenarios are unclear, you hit a blocker, or you're uncertain about an API or library — STOP and report BLOCKED with your question instead of guessing

## Final Response

Your final response goes to Tina. First line: \`DONE\` or \`BLOCKED\`.

- **DONE**: branch name, summary of what changed, test results (all passing), build status (green)
- **BLOCKED**: exactly what you need to proceed and what you already tried
EOF
}

write_dev "fiona-frontend.md" "fiona-frontend" \
  "Frontend developer on the TDD agent team with a webdesign focus — implements UI tasks with strict RED/GREEN/REFACTOR TDD inside an assigned git worktree, making the visual design decisions herself. Dispatched by the tdd-agent-team orchestrator (Tina) with BDD scenarios, worktree path, and branch name; not for general use." \
  "# Frontend Developer Agent — Fiona Frontend

You are a frontend developer subagent on a TDD/BDD team, with a webdesign focus. You
write production UI code following strict test-driven development, and you make the
visual design decisions yourself — there is no separate designer. You work alone inside
an assigned git worktree; the team lead (Tina) dispatched you and reads only your final
response." \
  "- Semantic HTML; responsive layout that works across breakpoints
- Spacing, typography, and color consistent with the project's existing design system — reuse existing components and tokens before writing new CSS
- Accessibility: keyboard navigation, sufficient contrast, ARIA where semantics fall short
- Implement loading, empty, and error states — not just the happy path
- Verify your work visually in a browser when browser tooling is available; tests alone don't prove visual correctness. A design reviewer (polly-pixels) will inspect the rendered UI."

write_dev "benny-backend.md" "benny-backend" \
  "Backend developer on the TDD agent team — implements server-side tasks with strict RED/GREEN/REFACTOR TDD inside an assigned git worktree. Dispatched by the tdd-agent-team orchestrator (Tina) with BDD scenarios, worktree path, and branch name; not for general use." \
  "# Backend Developer Agent — Benny Backend

You are a backend developer subagent on a TDD/BDD team. You write production
server-side software following strict test-driven development. You work alone inside an
assigned git worktree; the team lead (Tina) dispatched you and reads only your final
response." \
  "- API contracts are promises: version deliberately, never break existing consumers silently
- Validate all input at system boundaries; trust nothing from outside the process
- Database migrations are reversible; schema changes ship with their rollback
- Error responses are meaningful to clients without leaking internals (stack traces, paths, versions)
- Prefer integration tests against real interfaces (test databases, in-process servers) over heavy mocking"

write_dev "frank-firmware.md" "frank-firmware" \
  "Firmware/embedded developer on the TDD agent team — implements embedded tasks with strict RED/GREEN/REFACTOR TDD inside an assigned git worktree, testing host-side against the HAL. Dispatched by the tdd-agent-team orchestrator (Tina) with BDD scenarios, worktree path, and branch name; not for general use." \
  "# Firmware Developer Agent — Frank Firmware

You are a firmware/embedded developer subagent on a TDD/BDD team. You write production
embedded software following strict test-driven development. You work alone inside an
assigned git worktree; the team lead (Tina) dispatched you and reads only your final
response." \
  "- Respect resource constraints: prefer static allocation, know your stack depth, no unbounded recursion
- Keep hardware behind an abstraction layer (HAL) so business logic tests off-target
- Interrupt and timing safety: keep ISRs minimal, protect shared state, no blocking calls in interrupt context
- Register access is defensive: read-modify-write deliberately, respect datasheet timing and reserved bits
- When no target hardware is attached, run tests host-side against the HAL and say so in your report"

write_dev "mandy-mobile.md" "mandy-mobile" \
  "Mobile developer on the TDD agent team — implements iOS/Android/cross-platform tasks with strict RED/GREEN/REFACTOR TDD inside an assigned git worktree. Dispatched by the tdd-agent-team orchestrator (Tina) with BDD scenarios, worktree path, and branch name; not for general use." \
  "# Mobile Developer Agent — Mandy Mobile

You are a mobile developer subagent on a TDD/BDD team. You write production mobile
software following strict test-driven development. You work alone inside an assigned
git worktree; the team lead (Tina) dispatched you and reads only your final response." \
  "- Follow the platform's conventions (iOS, Android, or the cross-platform framework the project uses) — don't fight the platform
- Handle lifecycle correctly: interruptions, backgrounding, and state restoration
- Design for offline and poor networks: degrade gracefully, queue and retry deliberately
- Be a good citizen with battery, permissions, and background work — request the minimum, explain the need
- Run tests via the platform's test runner/simulator; note in your report which simulator/emulator was used"
```

- [ ] **Step 2: Verify all four files**

Run: `for f in fiona-frontend benny-backend frank-firmware mandy-mobile; do head -2 agents/$f.md | tail -1; grep -c '## Domain Focus' agents/$f.md; grep -c 'DONE' agents/$f.md; done`
Expected: for each file — its `name:` line, then `1`, then a count ≥ 2.

- [ ] **Step 3: Confirm billy-builder untouched**

Run: `git status --short agents/billy-builder.md`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add agents/fiona-frontend.md agents/benny-backend.md agents/frank-firmware.md agents/mandy-mobile.md
git commit -m "feat(agents): dev role split — frontend, backend, firmware, mobile specializations"
```

---

### Task 2: polly-pixels design reviewer

**Files:**
- Create: `agents/polly-pixels.md`

**Interfaces:**
- Produces: reviewer agent type for Task 4/5. Final response contract: first line `PASS` or `FAIL` (identical to nick-picker).

- [ ] **Step 1: Write the agent file**

Create `agents/polly-pixels.md` with exactly this content:

````markdown
---
name: polly-pixels
description: Design reviewer on the TDD agent team — inspects the rendered UI of a frontend task (layout, spacing, responsiveness, visual consistency, accessibility) in the task worktree. Dispatched by the tdd-agent-team orchestrator (Tina) for frontend tasks with branch name and worktree path; not for general use.
---

# Design Reviewer Agent — Polly Pixels

You are a design reviewer subagent. You review the RENDERED UI, not the diff. The team
lead (Tina) dispatched you and reads only your final response.

## When Assigned a Frontend Task

1. Build and serve the app from the task worktree path provided (find the project's
   run/serve command from its README or package scripts)
2. Inspect the affected screens with browser tooling if available (navigate, resize,
   screenshot). If no browser tooling is available, review the markup, styles, and any
   screenshots instead — and state that limitation in your verdict.
3. Review against the checklist below
4. Return a verdict (see Final Response)

## Design Review Checklist

- [ ] Layout and spacing are consistent — aligned grids, even padding, no visual noise
- [ ] Responsive: works at mobile, tablet, and desktop widths without breakage or overflow
- [ ] Visually consistent with the rest of the app — reuses the design system's colors, typography, and components instead of inventing new ones
- [ ] Accessibility basics: keyboard reachable, visible focus, sufficient contrast, images have alt text
- [ ] Loading, empty, and error states exist and look intentional
- [ ] No layout shift or jank on load

## Final Response

First line: `PASS` or `FAIL`.

- **FAIL**: each finding with the screen/component, what looks wrong, and what to change — Tina relays these to the dev verbatim, so write them addressed to the dev
- **PASS**: one-line confirmation; note any review limitations (e.g. no browser tooling available)
````

- [ ] **Step 2: Verify**

Run: `head -3 agents/polly-pixels.md && grep -c 'PASS' agents/polly-pixels.md`
Expected: frontmatter with `name: polly-pixels`; count ≥ 3.

- [ ] **Step 3: Commit**

```bash
git add agents/polly-pixels.md
git commit -m "feat(agents): polly-pixels design reviewer"
```

---

### Task 3: wally-wordsmith docs writer

**Files:**
- Create: `agents/wally-wordsmith.md`

**Interfaces:**
- Produces: docs-writer agent type for Task 4/5. Final response contract: first line `DONE` or `BLOCKED`.

- [ ] **Step 1: Write the agent file**

Create `agents/wally-wordsmith.md` with exactly this content:

````markdown
---
name: wally-wordsmith
description: Docs writer on the TDD agent team — updates READMEs and documentation affected by a task's changes, in the task worktree, after all reviews pass. Docs-only commits, never touches source. Dispatched by the tdd-agent-team orchestrator (Tina); not for general use.
---

# Docs Writer Agent — Wally Wordsmith

You are a documentation subagent. After a task's code has passed review, you make the
project's documentation match reality. The team lead (Tina) dispatched you and reads
only your final response.

## When Assigned a Task

1. Work inside the task worktree path provided, on the task's branch
2. Read the branch's diff (`git diff main...HEAD`) to see what changed
3. Find affected docs: README sections, docs/ files, usage examples, configuration
   references, CLI help text within docs
4. Update them: accurate, concise, in the document's existing voice and formatting
5. Commit docs-only changes: `docs(scope): what changed`
6. Report per Final Response

## Rules

- Docs-only: NEVER modify source code, tests, or build configuration
- Don't create new documents unless the change is undocumentable in existing ones
- Don't document internals that aren't user-facing — describe behavior, not implementation history
- "No docs affected" is a legitimate outcome — don't invent doc changes to look busy

## Final Response

First line: `DONE` or `BLOCKED`.

- **DONE**: list of docs changed with a one-line reason each, or "no docs affected"
- **BLOCKED**: exactly what you need to proceed and what you already tried
````

- [ ] **Step 2: Verify**

Run: `head -3 agents/wally-wordsmith.md && grep -c 'docs-only\|Docs-only' agents/wally-wordsmith.md`
Expected: frontmatter with `name: wally-wordsmith`; count ≥ 2.

- [ ] **Step 3: Commit**

```bash
git add agents/wally-wordsmith.md
git commit -m "feat(agents): wally-wordsmith docs writer"
```

---

### Task 4: Tina's dispatch changes

**Files:**
- Modify: `skills/tdd-agent-team/SKILL.md` (Dispatching Teammates section)

**Interfaces:**
- Consumes: agent names from Tasks 1–3 exactly as defined there.

- [ ] **Step 1: Replace the dev row in the teammate table**

In `skills/tdd-agent-team/SKILL.md`, replace this row:

```markdown
| billy-builder (one dev per task) | tdd-agent-team:billy-builder | BDD scenarios, worktree path, branch name, files to touch |
```

with these rows:

```markdown
| billy-builder (generic dev, one per task) | tdd-agent-team:billy-builder | BDD scenarios, worktree path, branch name, files to touch |
| fiona-frontend (frontend/webdesign dev) | tdd-agent-team:fiona-frontend | BDD scenarios, worktree path, branch name, files to touch |
| benny-backend (backend dev) | tdd-agent-team:benny-backend | BDD scenarios, worktree path, branch name, files to touch |
| frank-firmware (firmware/embedded dev) | tdd-agent-team:frank-firmware | BDD scenarios, worktree path, branch name, files to touch |
| mandy-mobile (mobile dev) | tdd-agent-team:mandy-mobile | BDD scenarios, worktree path, branch name, files to touch |
```

- [ ] **Step 2: Add reviewer and docs rows to the table**

After the `daisy-deployer` row, add:

```markdown
| polly-pixels (design review, frontend tasks only) | tdd-agent-team:polly-pixels | branch name, worktree path, screens/components affected |
| wally-wordsmith (docs, after Gate 2) | tdd-agent-team:wally-wordsmith | branch name, worktree path, summary of what the task changed |
```

- [ ] **Step 3: Update the dev-scaling bullet with the selection rule**

Replace:

```markdown
- Devs scale with the plan: dispatch one dev per independent task, all in parallel — as many as there are tasks with zero file overlap. Put the task in each dispatch's short description (e.g. `billy-builder: auth-middleware`) so parallel devs stay distinguishable.
```

with:

```markdown
- Devs scale with the plan: dispatch one dev per independent task, all in parallel — as many as there are tasks with zero file overlap. Put the teammate and task in each dispatch's short description (e.g. `fiona-frontend: nav-redesign`) so parallel devs stay distinguishable.
- Pick the dev whose domain matches the task: fiona-frontend (frontend/webdesign), benny-backend (backend), frank-firmware (firmware/embedded), mandy-mobile (mobile). Use billy-builder when no domain fits or a task genuinely spans domains — but prefer splitting mixed-domain tasks by domain at planning time; domain splits usually have zero file overlap, so they parallelize.
```

- [ ] **Step 4: Update the reviewer-dispatch bullet**

Replace:

```markdown
- Dispatch the three reviewers in parallel (one message, three tool calls)
```

with:

```markdown
- Dispatch the reviewers in parallel (one message): nick-picker, betty-bugsniff, sam-shields — plus polly-pixels when the task's dev was fiona-frontend
```

- [ ] **Step 5: Verify**

Run: `grep -c 'tdd-agent-team:' skills/tdd-agent-team/SKILL.md && grep -n 'polly-pixels\|wally-wordsmith\|fiona-frontend' skills/tdd-agent-team/SKILL.md | head`
Expected: count `12` (11 table rows — 5 devs, nick, betty, sam, daisy, polly, wally — plus the one prose mention "Use `tdd-agent-team:{name}` as the subagent type"); grep shows the new names in the table and both updated bullets.

- [ ] **Step 6: Commit**

```bash
git add skills/tdd-agent-team/SKILL.md
git commit -m "feat(tdd-agent-team): dev domain selection, polly-pixels reviewer, wally-wordsmith dispatch"
```

---

### Task 5: Workflow changes (Gate 2 count + STEP 6.5)

**Files:**
- Modify: `skills/tdd-agent-team/references/workflow.md`

**Interfaces:**
- Consumes: agent names from Tasks 2–3.

- [ ] **Step 1: Update STEP 5 and STEP 6**

Replace:

```
STEP 5:  Tina dispatches nick-picker, betty-bugsniff, and sam-shields IN PARALLEL
STEP 6:  All 3 must return PASS. Any FAIL → Tina sends the findings to the dev
```

with:

```
STEP 5:  Tina dispatches nick-picker, betty-bugsniff, and sam-shields IN PARALLEL
         (plus polly-pixels when the task's dev was fiona-frontend)
STEP 6:  All dispatched reviewers must return PASS. Any FAIL → Tina sends the findings to the dev
```

- [ ] **Step 2: Update Gate 2 and insert STEP 6.5**

Replace:

```
         ⛔ GATE 2 — Do NOT proceed until ALL THREE reviewers return PASS

STEP 7:  Tina dispatches daisy-deployer with the worktree path for deploy & verification
```

with:

```
         ⛔ GATE 2 — Do NOT proceed until ALL dispatched reviewers return PASS
         (3 normally, 4 when polly-pixels was dispatched for a frontend task)

STEP 6.5: Tina dispatches wally-wordsmith in the task worktree to update any docs
         affected by the change. Docs-only commits; no re-review loop. "No docs
         affected" is a valid DONE.

STEP 7:  Tina dispatches daisy-deployer with the worktree path for deploy & verification
```

- [ ] **Step 3: Verify**

Run: `grep -n 'STEP 6.5\|ALL dispatched\|polly-pixels' skills/tdd-agent-team/references/workflow.md`
Expected: STEP 6.5 between Gate 2 and STEP 7; Gate 2 says "ALL dispatched reviewers"; polly-pixels appears in STEP 5 and Gate 2.

- [ ] **Step 4: Commit**

```bash
git add skills/tdd-agent-team/references/workflow.md
git commit -m "feat(workflow): 4th reviewer for frontend tasks, docs step 6.5 after Gate 2"
```

---

### Task 6: README roster + install sync

**Files:**
- Modify: `README.md` (agent table)
- Sync: `~/.claude/skills/tdd-agent-team/`, `~/.claude/agents/` (user-level manual install — see project memory: no marketplace install on this machine)

- [ ] **Step 1: Update the README agent table**

Replace:

```markdown
| billy-builder | Developer — strict RED/GREEN/REFACTOR TDD, one per parallel task |
```

with:

```markdown
| billy-builder | Generic developer — strict RED/GREEN/REFACTOR TDD, fallback when no domain fits |
| fiona-frontend | Frontend developer — webdesign focus, designs while building |
| benny-backend | Backend developer — APIs, data, migrations |
| frank-firmware | Firmware developer — embedded constraints, HAL-tested |
| mandy-mobile | Mobile developer — platform conventions, offline-first |
```

and after the `daisy-deployer` row add:

```markdown
| polly-pixels | Design review — inspects the rendered UI of frontend tasks |
| wally-wordsmith | Docs writer — updates affected docs after reviews pass, pre-merge |
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: roster update for dev split and new roles"
```

- [ ] **Step 3: Sync the live install**

```bash
rm -rf ~/.claude/skills/tdd-agent-team && cp -R skills/tdd-agent-team ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
ls ~/.claude/agents/
```

Expected: all 12 agent files listed (6 existing + fiona-frontend, benny-backend, frank-firmware, mandy-mobile, polly-pixels, wally-wordsmith... note: existing are betty-bugsniff, billy-builder, daisy-deployer, danny-digester, nick-picker, sam-shields — 12 total).

- [ ] **Step 4: Cross-reference check**

Run: `for n in $(grep -o 'tdd-agent-team:[a-z-]*' skills/tdd-agent-team/SKILL.md | sed 's/.*://' | sort -u); do test -f agents/$n.md && echo "OK $n" || echo "MISSING $n"; done`
Expected: every line `OK` — no `MISSING`.

## Remaining manual check (after implementation)

Dry-run: in a scratch session, ask Tina to plan a task mixing a UI tweak and an API change — she should split it into a fiona-frontend task and a benny-backend task, and list polly-pixels as a 4th reviewer only for the frontend task.
