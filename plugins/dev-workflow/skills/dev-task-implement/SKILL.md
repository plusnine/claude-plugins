---
name: dev-task-implement
description: Investigate codebase, resolve spec gaps, and implement a single coarse task from dev-spec-breakdown output.
---

## Purpose

Takes a coarse task prompt from dev-spec-breakdown, investigates the codebase scoped to that task,
resolves spec gaps, and applies implementation code with user approval.

## Input

- Task prompt at `claude-output/{id}/dev-spec-breakdown/tasks/{nn}-{name}.md`
- Target codebase

## Output

Written to `claude-output/{id}/dev-task-implement/`:

- `{nn}-spec-gaps.md` — Spec gaps found during investigation (only when gaps exist). Format: `references/spec-gaps-format.md`
- `{nn}-plan.md` — Implementation plan written after gap resolution and approved by user. Format: `references/implementation-plan-format.md`
- `{nn}-progress.md` — Tracks per-file application status during Step 5. Transient: renamed to `{nn}-done.md` on completion.
- `{nn}-done.md` — Renamed from `{nn}-progress.md` after user approves completion. Presence indicates task is complete.

### progress.md Format

```
# Progress — Task {nn}: {task name}

| # | File | Status |
|---|------|--------|
| 1 | `path/to/file.kt` | ⏳ Pending |
| 2 | `path/to/new_file.kt` | ✅ Applied |
| 3 | `path/to/other.kt` | ❌ Rejected |
```

## Sub-task Split Criteria

After gap resolution, evaluate whether the task scope fits one atomic implementation:
- Split if the task spans multiple independent concerns that can be implemented and tested separately
- Split if a single logical review of the plan is not feasible (too many unrelated moving parts)
- Do NOT split based solely on line count or file count — large changes concentrated in one concern are acceptable as a single sub-task
- Exception: identical operations repeated across files (e.g., rename, import changes) are always a single sub-task regardless of scope

## Spec Gap Priority

| Priority | Meaning | Effect |
|----------|---------|--------|
| 🔴 Required | Must be resolved before implementation | Blocks implementation |
| 🟡 Recommended | Recommended to resolve, may proceed | Does not block |
| ⚪ Optional | Optional | Does not block |
