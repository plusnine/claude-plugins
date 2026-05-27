---
description: Decompose spec-review or bugfix investigation artifacts into coarse tasks. No code investigation — business-level decomposition only.
argument-hint: "{id}"
---

All file output must be written in English regardless of content origin.
User-facing output (messages, previews, etc.) must be in the language specified by `language` in `~/.claude/settings.json` (fallback: English). This includes approval preview presentations (plan previews, diff summaries, PR previews): file content stays in English, but all labels, section headers, and descriptions presented to the user must follow the language setting.

## Invocation

`/spec-breakdown {id}`

## Input

- Spec-review artifacts at `claude-output/{id}/spec-review/`, OR
- Bugfix investigation report at `claude-output/{id}/bugfix/investigation-report.md` (Status: FINAL)

## Output

Written to `claude-output/{id}/spec-breakdown/`:
- `plan.md` — Coarse task list (Phase 1). Format: `references/plan-format.md`
- `tasks/01-{name}.md` — Coarse task prompt files (Phase 2). Format: `references/task-format.md`
- `tasks/02-{name}.md`
- ...

## Checkpoint

On re-run, resume position is determined by existing files.

**Precondition**: any resume position below assumes `claude-output/{id}/meta.md` exists per "Resolve target repository". If `meta.md` is missing, run "Resolve target repository" before resuming from any other state.

| State | Resume from |
|-------|-------------|
| `tasks/` exists and file count matches task count in `plan.md` | Already complete |
| `tasks/` exists but file count is less than task count in `plan.md` | Phase 2 (generate remaining task files) |
| `plan.md` exists & Status: APPROVED | Phase 2 |
| Otherwise | Phase 1 |

## Process

### Resolve target repository (precondition)

If `claude-output/{id}/meta.md` does not exist, run the detection algorithm in `../../shared/references/meta-format.md` ("Detection algorithm") and write meta.md.

If meta.md exists, validate it per `../../shared/references/meta-format.md` ("Validation"). On validation failure, exit with an error in the user's configured language and require manual correction before re-run.

This section runs before Phase 1.

### Phase 1: Coarse Task Decomposition

Resolve input source in priority order:
1. `claude-output/{id}/spec-review/source.md` — if found, source-type is `spec-review`
2. `claude-output/{id}/bugfix/investigation-report.md` (Status: FINAL only; treat missing Status as DRAFT) — if found, source-type is `bugfix`

If neither exists → exit with message:
> No input source found. Run one of the following first:
> - `/dev-workflow:spec-review {spec-url-or-path}` — to start from specification review
> - `/dev-workflow:bugfix {id}` — to start from bug investigation

If source-type is `spec-review`:
- Read `source.md` as the primary input.
- Read `claude-output/{id}/spec-review/review.md` and check the Verdict:
  - `FAIL`: stop and prompt the user to resolve all 🔴 Required items in review.md before proceeding.
  - `CONDITIONAL PASS`: warn the user that 🟡 Recommended items remain unresolved, then ask whether to proceed.
  - `PASS` or review.md does not exist: proceed.
- Use `review.md` as supplementary reference.

If source-type is `bugfix`:
- Read `investigation-report.md` as the primary input. Decompose the "What to fix" section.
- `review.md` check is skipped (not applicable for bugfix flow).

Decompose requirements into coarse tasks based on business intent alone — no code investigation.
Apply `spec-breakdown` skill criteria when decomposing tasks.
Present `plan.md` to the user for approval before proceeding to Phase 2.
If the user requests changes (e.g., merge or split tasks): update `plan.md` and re-present for approval. Repeat until approved.

If a Phase 1 re-approval reduces the task count below the number of already-existing `tasks/{nn}-*.md` files, those excess files are orphans. The command does not auto-delete them. List the orphan files by name to the user and require manual deletion before proceeding. Do not enter Phase 2 until the file count in `tasks/` matches the task count in `plan.md`.

### Phase 2: Task Prompt Generation

Generate one prompt file per task in `tasks/`. Filename: `{nn}-{kebab-case-task-name}.md` where `{nn}` is the task number from plan.md and `{kebab-case-task-name}` is derived from the task name.
When resuming partial Phase 2: compare the task list in `plan.md` against existing files in `tasks/` and generate only the missing ones.
