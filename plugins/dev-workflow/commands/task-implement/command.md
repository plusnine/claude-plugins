---
description: Orchestrates investigation, gap resolution, sub-task evaluation, and implementation for a single task. Owns all file writes.
argument-hint: "{id} {nn}"
---

All file output must be written in English regardless of content origin.
User-facing output (messages, previews, etc.) must be in the language specified by `language` in `~/.claude/settings.json` (fallback: English). This includes approval preview presentations (plan previews, diff summaries, PR previews): file content stays in English, but all labels, section headers, and descriptions presented to the user must follow the language setting.

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

Before invoking the agent, load Index Hints from code-map per `../../shared/references/code-map-format.md`:

1. Resolve `{repo-name}` per `../../shared/references/code-map-format.md` (`basename $(git rev-parse --show-toplevel)` lowercased). If not in a git repo: skip the hints path entirely
2. If `claude-output/_index/{repo-name}/code-map.md` exists:
   a. Extract keywords from the task prompt (task title tokens + prominent noun phrases from "What to implement"). Expand with semantic equivalents — synonyms, related terms, alternative namings the codebase may use (query expansion). Leverage CLAUDE.md for domain vocabulary
   b. Filter entries by case-insensitive substring match on `concept` column against any keyword (base or expanded)
   c. Deduplicate: for each duplicated concept, keep the entry with the most recent `verified_at` (by git commit recency). Tiebreaker for same verified_at: keep the last-written (bottom-most) row. Remove older duplicates from the file
   d. For each remaining candidate entry, verify:
      - All paths in `starting_points` exist (remove entry from file if any path missing)
      - `git diff {verified_at}..HEAD -- {starting_points}`: success + no diff → high confidence; success + diff → lower confidence; command failure (verified_at unreachable) → remove entry from file
   e. Pass verified entries to the agent as Index Hints (markdown table format, see `../../shared/references/code-map-format.md` "Agent integration")
3. If code-map does not exist or no hints survived: skip the hints path — no change to agent invocation

Surface to user a one-line summary after the read phase: `Index: N hints used (M high, K lower), S stale removed`. If no hints used, `Index: no matches`.

Invoke `task-implement:investigate` agent. Pass:
- The full content of the task prompt file
- The path to the project root `CLAUDE.md` (agent will read it directly for codebase context)
- Index Hints (if any)

If the Affected Files list is empty: present the findings to the user and propose skipping this task.
On confirmation: write `{nn}-skipped.md` with reason and exit.
On rejection: ask the user how to proceed before continuing.

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

### Step 3.5: Update code-map

After Step 3 (investigation validated by the user, either directly or via gap resolution), append a new entry to `claude-output/_index/{repo-name}/code-map.md` per `../../shared/references/code-map-format.md`. Resolve `{repo-name}` fresh per code-map-format.md (`basename $(git rev-parse --show-toplevel)` lowercased; skip this entire step if not in a git repo):

- `concept`: strip the `{nn}-` prefix from `tasks/{nn}-{task-name}.md` filename, then normalize (kebab-case → space-separated, lowercase, trimmed). Example: `01-add-dark-mode.md` → `"add dark mode"`
- `starting_points`: agent's reported Starting Points (pipe-joined, priority order preserved)
- `verified_at`: current `git rev-parse --short HEAD`

Create `claude-output/_index/{repo-name}/` if it does not exist. Append at end of file (no merge with existing rows; dedup handled at read time).

Skip this step if the agent returned no Starting Points (unlikely, but defensive).

Surface to user after append: `Index: appended '{concept}' → {starting_points} @ {verified_at}`. If skipped, state why (e.g., `Index: skipped — not a git repo` / `Index: skipped — no starting points`).

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

After all file changes are applied: notify the user that changes are complete and proceed to Step 6. The user may request a pause before the Step 6.5 preview if they need time to verify.

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
6. After Create draft PR completes (all rows in `{nn}-progress.md` ✅), auto-rename `{nn}-progress.md` → `{nn}-done.md`.

## Checkpoint

On re-run, resume position is determined by existing files:

| State | Resume from |
|-------|-------------|
| `{nn}-done.md` or `{nn}-skipped.md` exists | Already complete |
| `{nn}-progress.md` exists & all rows ✅ | Auto-completion interrupted before rename — rename to `{nn}-done.md` |
| `{nn}-progress.md` exists | Resume from the first ⏳ row in progress.md (branch → file changes → commit → push → create PR) |
| `{nn}-plan.md` exists | Step 5 (present plan for approval, write progress.md, start applying) |
| `{nn}-spec-gaps.md` exists & Status: OPEN | Step 3 (gap resolution loop) |
| `{nn}-spec-gaps.md` exists & Status: RESOLVED | Step 2 (re-investigate to restore Affected Files); see "Resumption after resolved gaps" below |
| Otherwise | Step 0 (prerequisites), then Step 2 (investigate) |

### Resumption after resolved gaps

When `{nn}-spec-gaps.md` exists with Status: RESOLVED, re-invoke Step 2 to restore Affected Files, then evaluate any new gaps:

- 🔴 Required gaps found → set `{nn}-spec-gaps.md` Status back to OPEN, go to Step 3
- Only 🟡 Recommended or ⚪ Optional gaps → present to user; go to Step 4 on proceed, Step 3 on resolve
- No new gaps → go to Step 4

### Note on code-map writes (Step 3.5)

Step 3.5 does not produce a persistent state file. Re-runs after resumption may append duplicate entries to `_index/{repo-name}/code-map.md`. Duplicates are harmless: dedup happens at read time (most-recent `verified_at` wins; older rows are removed on next read per `../../shared/references/code-map-format.md` reader contract).
