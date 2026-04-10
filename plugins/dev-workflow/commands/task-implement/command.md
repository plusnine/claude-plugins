---
description: Orchestrates investigation, gap resolution, sub-task evaluation, and implementation for a single task. Owns all file writes.
argument-hint: "{id} {nn}"
---

All file output must be written in English regardless of content origin.
User-facing output (messages, previews, etc.) must be in the language specified by `language` in `~/.claude/settings.json` (fallback: English).

## Invocation

`/task-implement {id} {nn}`

## Process

### Step 0: Prerequisites

Verify a reference document exists, in priority order:
1. `claude-output/{id}/spec-review/source.md`
2. `claude-output/{id}/bugfix/investigation-report.md` (Status: FINAL; treat missing Status as DRAFT)

If neither exists → exit with message:
> No reference document found. Run one of the following first:
> - `/dev-workflow:spec-review {spec-url-or-path}` — to start from specification review
> - `/dev-workflow:bugfix {id}` — to start from bug investigation

### Step 1: Load task

Read `claude-output/{id}/spec-breakdown/tasks/{nn}-*.md`.

If the task lists Prerequisites, verify each prerequisite task has a corresponding `{nn}-done.md`.
If any prerequisite is incomplete → exit with message:
> Prerequisite task {nn} is not yet complete. Run `/dev-workflow:task-implement {id} {nn}` first.

### Step 2: Investigate

Invoke `task-implement:investigate` agent. Pass:
- The full content of the task prompt file
- The path to the project root `CLAUDE.md` (the one containing "Investigation Entry Points"; agent will read it directly)

If the Affected Files list is empty: present the findings to the user and propose skipping this task.
On confirmation: write `{nn}-skipped.md` with reason and exit.
On rejection: ask the user how to proceed before continuing.

### Step 2b: Register new entry points (unknown area only)

If the investigate agent reports `Area: unknown`:
1. Present proposed entry points to user
2. On approval: append to the project root `CLAUDE.md`'s "Investigation Entry Points" section (create the section if it does not exist) using this format:
   ```
   ### {subsystem name} (e.g. "list-screen", "detail-screen", or a new name if the area is distinct)
   #### {area name}
   Entry files (read in this order):
   1. `{file}` — {role}

   Fixed change targets:
   - {fixed change targets if identified, otherwise "TBD"}
   ```
   If the new area fits under an existing subsystem (H3), append under it instead of creating a new H3.
3. On rejection: proceed without registering

### Step 3: Gap resolution loop

If spec gaps exist:
If only 🟡 Recommended or ⚪ Optional gaps exist (no 🔴 Required): write `claude-output/{id}/task-implement/{nn}-spec-gaps.md` with Status: RESOLVED and present to user. Skip to Step 4.
If 🔴 Required gaps exist:
1. Write `claude-output/{id}/task-implement/{nn}-spec-gaps.md` with Status: OPEN
2. Present gaps to user
3. User provides resolutions → update Resolution Log in `{nn}-spec-gaps.md`
4. If the user's resolution requires verifying new code areas: re-invoke `task-implement:investigate` agent, passing the updated spec gaps content alongside the original task prompt
5. Repeat until no 🔴 Required gaps remain
6. Update `{nn}-spec-gaps.md` Status to RESOLVED.

### Step 4: Evaluate task size

Apply Sub-task Split Criteria from `task-implement/SKILL.md`:
- If split needed: propose sub-tasks to user → on confirmation, treat each as a separate implementation unit
- If ok: proceed

### Step 4.5: Create branch

Read `references/pr-guidelines.md` (Branch Naming section) before proceeding.

Verify the current branch matches the parent branch pattern `{prefix}/{id}` (e.g. `feature/PROJ-123`).
If it does not match → exit with message:
> Parent branch not found. Create and checkout the parent branch first (e.g. `git checkout -b feature/{id}`), then re-run this command.

If this is the only task (single task per `references/pr-guidelines.md` Branch Naming): skip branch creation and use the parent branch as the head branch. Proceed to Step 5.

Create the working branch per `references/pr-guidelines.md` Branch Naming. Inherit the prefix from the parent branch.
Present branch name to user → **Approval ①**.
If the branch already exists: confirm with user whether to reuse it.
Carry-over of uncommitted changes is acceptable; only implementation changes are committed.

### Step 5: Write plan and apply

Write `claude-output/{id}/task-implement/{nn}-plan.md`.
Present the plan to the user. Wait for explicit approval before proceeding.
On approval: Write `claude-output/{id}/task-implement/{nn}-progress.md` per `references/progress-format.md` — branch creation as row 1 (✅ if already done in Step 4.5), all files as ⏳ Pending, plus Commit / Push / Create draft PR rows as ⏳ Pending.
For each file change (apply in dependency order if Dependencies are specified in the plan):
1. Apply the change — requires user approval per the Write Safety Gate
2. On approval: update `{nn}-progress.md` → ✅ Applied
3. On rejection: revise the change based on user feedback and re-present. Repeat until the user approves (✅ Applied) or explicitly skips the file (⏭ Skipped).

After all file changes are applied: notify the user and **wait for verification completion**. Do not proceed until the user confirms.

### Step 6: Commit, push, and PR

Read `references/pr-guidelines.md` in full before proceeding with any sub-step.

If ⏭ Skipped entries exist: present them to the user before proceeding.

1. Analyze actual `git diff` and propose commit grouping per `references/pr-guidelines.md` Commit Guidelines.
2. Resolve base branch per `references/pr-guidelines.md` Base Branch Resolution.
3. Check for existing PR via `gh pr list --head {branch}`.
4. Check repository visibility via `gh repo view --json isPrivate`.
5. Present unified preview per `references/pr-guidelines.md` Unified Preview Format → **Approval ②**.
   User may request modifications to commit messages, file grouping, PR title/body via numbered instructions.
   On approval: execute commit → push → create/update draft PR sequentially.
   Update `{nn}-progress.md` rows (Commit → ✅, Push → ✅, Create draft PR → ✅) as each completes.
   User may request separation of push and PR creation steps.
6. Ask the user: "Mark this task as done?" → **Approval ③**.
   On explicit approval: rename `{nn}-progress.md` → `{nn}-done.md`.

## Checkpoint

On re-run, resume position is determined by existing files:

| State | Resume from |
|-------|-------------|
| `{nn}-done.md` exists | Already complete |
| `{nn}-skipped.md` exists | Already complete (skipped) |
| `{nn}-progress.md` exists & all ✅ | Step 6.6 (mark as done) |
| `{nn}-progress.md` exists & Push ✅, PR ⏳ | Step 6.5 (create PR) |
| `{nn}-progress.md` exists & Commit ✅, Push ⏳ | Step 6.5 (push) |
| `{nn}-progress.md` exists & files all ✅, Commit ⏳ | Step 6.1 (verification or unified preview) |
| `{nn}-progress.md` exists & file ⏳ remain | Step 5 (apply remaining files) |
| `{nn}-progress.md` exists & branch ⏳ | Step 4.5 (create branch) |
| `{nn}-plan.md` exists | Step 5 (present plan for approval, write progress.md, start applying) |
| `{nn}-spec-gaps.md` exists & Status: OPEN | Step 3 (gap resolution loop) |
| `{nn}-spec-gaps.md` exists & Status: RESOLVED | Step 2 (re-investigate to restore Affected Files; if Area: unknown, also run Step 2b), then evaluate new gaps: 🔴 Required → update Status to OPEN, go to Step 3; 🟡/⚪ → present to user, ask proceed or resolve, on proceed go to Step 4, on resolve go to Step 3; none → go to Step 4 |
| Otherwise | Step 0 (prerequisites), then Step 2 (investigate) |
