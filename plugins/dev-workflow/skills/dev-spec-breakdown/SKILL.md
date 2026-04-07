---
name: dev-spec-breakdown
description: Decompose spec-review artifacts into coarse tasks. No code investigation — business-level decomposition only.
---

## Purpose

Transform spec-review artifacts into a coarse task list and task prompts based on business requirements alone.
Code investigation is deferred to task-implement.

## Input

- Spec-review artifacts at `claude-output/{id}/dev-spec-review/`
  (user specifies `{id}` via `/dev-spec-breakdown {id}`)

## Output

Written to `claude-output/{id}/dev-spec-breakdown/`:

- `plan.md` — Coarse task list (Phase 1). Format: `references/plan-format.md`
- `tasks/01-{name}.md` — Coarse task prompt files (Phase 2). Format: `references/task-format.md`
- `tasks/02-{name}.md`
- ...

### Checkpoint

On re-run, resume position is determined by existing files:

| State | Resume from |
|-------|-------------|
| `tasks/` exists and file count matches task count in `plan.md` | Already complete |
| `tasks/` exists but file count is less than task count in `plan.md` | Phase 2 (generate remaining task files) |
| `plan.md` exists & Status: APPROVED | Phase 2 |
| Otherwise | Phase 1 |

## Process

### Phase 1: Coarse Task Decomposition

Read `claude-output/{id}/dev-spec-review/source.md` as the primary input.
If `source.md` does not exist: stop and prompt the user to run spec-review first.

Read `claude-output/{id}/dev-spec-review/review.md` and check the Verdict:
- `FAIL`: stop and prompt the user to resolve all 🔴 Required items in review.md before proceeding.
- `CONDITIONAL PASS`: warn the user that 🟡 Recommended items remain unresolved, then ask whether to proceed.
- `PASS` or review.md does not exist: proceed.

Use `review.md` in the same directory as supplementary reference to understand identified gaps and issues.
Decompose requirements into coarse tasks based on business intent alone — no code investigation.
Present `plan.md` to the user for approval before proceeding to Phase 2.
If the user requests changes (e.g., merge or split tasks): update `plan.md` and re-present for approval. Repeat until approved.

### Phase 2: Task Prompt Generation

Generate one prompt file per task in `tasks/`. Filename: `{nn}-{kebab-case-task-name}.md` where `{nn}` is the task number from plan.md and `{kebab-case-task-name}` is derived from the task name.
When resuming partial Phase 2: compare the task list in `plan.md` against existing files in `tasks/` and generate only the missing ones.

## Coarse Task Criteria

### Primary rule
A task is appropriately scoped if it can be described in a single sentence without "and".
- If not: split into separate tasks
- If yes: the task is valid even if it involves heavy internal processing with no user-visible outcome

### Split signals (detectable from spec text)
- Additive connectives (e.g. "also", "additionally" — or equivalent in the spec language) appear more than once → split candidate
- Sequential dependency language (e.g. "after X is done", "once X is complete" — or equivalent in the spec language) → split into before/after tasks
- Multiple distinct actors appear with independent goals (e.g., user + admin each triggering separate flows) → split candidate; actors cooperating in a single flow do not qualify

### Content rules
- Do NOT reference specific files or class names — those are determined by task-implement
- Exception: if a file/class is explicitly named in the spec, it may be referenced
