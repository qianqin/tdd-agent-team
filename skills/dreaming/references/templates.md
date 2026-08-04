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
