---
name: task-implement
description: Investigate codebase, resolve spec gaps, and implement a single coarse task from spec-breakdown output.
version: 0.2.0
model: sonnet
---

## Purpose

Takes a coarse task prompt from spec-breakdown, investigates the codebase scoped to that task,
resolves spec gaps, applies implementation code, and creates a draft pull request with user approval.

## Input

- Task prompt at `claude-output/{id}/spec-breakdown/tasks/{nn}-{name}.md`
- Reference document (one of, in priority order):
  - `claude-output/{id}/spec-review/source.md` (spec-review flow)
  - `claude-output/{id}/bugfix/investigation-report.md` (bugfix flow, Status: FINAL)
- Target codebase

## Output

Written to `claude-output/{id}/task-implement/`:

- `{nn}-spec-gaps.md` — Spec gaps found during investigation (only when gaps exist). Format: `references/spec-gaps-format.md`
- `{nn}-plan.md` — Implementation plan written after gap resolution and approved by user. Format: `references/implementation-plan-format.md`
- `{nn}-progress.md` — Tracks branch creation, file application, commit, push, and PR creation status. Format: `references/progress-format.md`. Transient: renamed to `{nn}-done.md` or `{nn}-skipped.md`.
- `{nn}-done.md` — Renamed from `{nn}-progress.md` after user approves completion. Presence indicates task is complete (implementation + PR created).
- `{nn}-skipped.md` — Task deemed unnecessary and skipped. Contains reason. No branch or PR created.

## PR Creation Flow

After implementation, the skill creates a draft PR. See `references/pr-guidelines.md` for:
- Branch naming and base branch resolution
- Commit guidelines (Semantic Commit Messages, size limits)
- PR content (template, title format, screenshot rules)
- Sensitive content handling

## Completion States

- `{nn}-done.md` — implementation complete + PR created
- `{nn}-skipped.md` — task not needed (reason documented), no branch or PR

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
