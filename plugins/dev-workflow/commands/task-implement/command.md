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

If `investigation-report.md` exists but Status is DRAFT (or missing): exit with message:
> Bugfix investigation is not yet finalized. Complete `/dev-workflow:bugfix {id}` through Step 2c first.

### Resolve target repository (precondition)

If `claude-output/{id}/meta.md` does not exist, run the detection algorithm in `../../shared/references/meta-format.md` ("Detection algorithm") and write meta.md.

If meta.md exists, validate it per `../../shared/references/meta-format.md` ("Validation"). On validation failure, exit with an error in the user's configured language and require manual correction before re-run.

This section runs between Step 0 and Step 1. All subsequent git/gh/glab operations follow the contract in `meta-format.md` "Reading".

### Step 1: Load task

Read `claude-output/{id}/spec-breakdown/tasks/{nn}-*.md`.

If the task lists Prerequisites, verify each prerequisite task has a corresponding `{nn}-done.md`.
If any prerequisite is incomplete → exit with message:
> Prerequisite task {nn} is not yet complete. Run `/dev-workflow:task-implement {id} {nn}` first.

### Step 2: Investigate

Before invoking the agent, load Index Hints from code-map per `../../shared/references/code-map-format.md` (sections: `{repo-name}` resolution, Read policy, Agent integration). Skip if not in a git repo.

Surface to user after the read phase: `Index: N hints used (M high, K lower), S stale removed`. If no hints used, `Index: no matches`.

Append a metrics line per `code-map-format.md` Metrics Log section (read trigger).

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

After Step 3 (investigation validated by the user), run the Write pipeline per `../../shared/references/code-map-format.md` Write policy. Skip entirely if not in a git repo.

1. **v1 legacy detection** (first v2 write only): if `claude-output/_index/{repo-name}/code-map.md` exists, present a rename plan to the user via the Write Safety Gate per `code-map-format.md` "v1 → v2 migration": `code-map.md` → `code-map.v1.deprecated.md` (timestamped suffix on collision). Proceed with the rename on approval; if rejected, continue with the v2 write leaving the v1 file untouched.
2. **Extract, validate, and append** per `code-map-format.md` Write pipeline (six layers: extraction, line structure, JSON syntax, schema, semantic, verified_at injection). The source is the `### Code Map Entry` block at the end of the investigate agent's reply. On layer 1-5 failure, retry exactly once per Retry protocol. On second failure, surface the rejection reason and skip the append (downstream steps continue).
3. **Skip (no agent block)**: if the agent response contains no `### Code Map Entry` heading, treat as graceful skip — surface `Index: skipped — no entry block` and continue.

Surface outcomes:
- Appended: `Index: appended '{concept}' → {N entries} @ {verified_at}`
- Skipped (no git repo): `Index: skipped — not a git repo`
- Skipped (no block): `Index: skipped — no entry block`
- Rejected (validation failed after retry): `Index: write rejected — {final reason}`

Append a metrics line per `code-map-format.md` Metrics Log section (write trigger) reflecting the outcome.

### Step 4: Evaluate task size

Apply Sub-task Split Criteria from `task-implement/SKILL.md`:
- If split needed: propose sub-tasks to user → on confirmation, treat each as a separate implementation unit
- If ok: proceed

### Step 4.5: Create branch

Read `references/pr-guidelines.md` Branch Naming section for rules. This step owns only orchestration — all naming/verification rules live in that file.

Verify the current branch is the parent branch per pr-guidelines.md Branch Naming. If not → exit with message:
> Parent branch not found in `{target-repository-path}`. Create and checkout the parent branch first (e.g. `git -C "{target-repository-path}" checkout -b feature/{id}`), then re-run this command.

If pr-guidelines.md Branch Naming resolves to "use parent branch directly" (single-task case): skip branch creation. Proceed to Step 5.

Otherwise, create the working branch per pr-guidelines.md Branch Naming.
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
3. Check for existing PR/MR on the same head branch using the platform's hosting CLI or REST API (e.g. `gh pr list --head {branch}` for GitHub, `glab mr list --source-branch {branch}` for GitLab, Bitbucket REST API for Bitbucket).
4. Check repository visibility using the platform's hosting CLI or REST API (e.g. `gh repo view --json isPrivate` for GitHub, `glab repo view` for GitLab, Bitbucket REST API for Bitbucket).
5. Present unified preview per `references/pr-guidelines.md` Unified Preview Format → **Approval ②**.
   User may request modifications to commit messages, file grouping, PR title/body via numbered instructions.
   On approval: execute commit → push → create/update draft PR/MR sequentially.
   Update `{nn}-progress.md` rows (Commit → ✅, Push → ✅, Create draft PR/MR → ✅) as each completes.
   User may request separation of push and PR/MR creation steps.
6. After Create draft PR/MR completes (all rows in `{nn}-progress.md` ✅), auto-rename `{nn}-progress.md` → `{nn}-done.md`.

On failure of any sub-step (commit / push / PR/MR create):

1. Keep the failed row as ⏳ in `{nn}-progress.md`.
2. Append a brief error note (executed command + stderr verbatim) to `{nn}-progress.md`.
3. Surface the error to the user.
4. Exit.

Do **not**:
- Auto-retry the failed sub-step.
- Propose, attempt, or describe a fix (e.g., do not run `git pull --rebase`, `git push --force`, change git config, open a PR via the web UI, or modify the working tree).
- Modify any other rows in `{nn}-progress.md`.
- Roll back applied file changes or delete the working branch.

The user is responsible for diagnosing the failure, resolving it manually outside the command, and re-running. Resumption picks up at the failed ⏳ row per the Checkpoint table.

## Checkpoint

On re-run, resume position is determined by existing files.

**Precondition**: any resume position below assumes `claude-output/{id}/meta.md` exists per "Resolve target repository". If `meta.md` is missing, run "Resolve target repository" before resuming from any other state.

| State | Resume from |
|-------|-------------|
| `{nn}-done.md` or `{nn}-skipped.md` exists | Already complete |
| `{nn}-progress.md` exists & all rows ✅ | Auto-completion interrupted before rename — rename to `{nn}-done.md` |
| `{nn}-progress.md` exists | Resume from the first ⏳ row in progress.md (branch → file changes → commit → push → create PR/MR) |
| `{nn}-plan.md` exists | Step 5 (present plan for approval, write progress.md, start applying) |
| `{nn}-spec-gaps.md` exists & Status: OPEN | Step 3 (gap resolution loop) |
| `{nn}-spec-gaps.md` exists & Status: RESOLVED | Step 2 (re-investigate to restore Affected Files); see "Resumption after resolved gaps" below |
| Otherwise | Step 0 (prerequisites), then Step 2 (investigate) |

### Resumption after resolved gaps

When `{nn}-spec-gaps.md` exists with Status: RESOLVED, re-invoke Step 2 to restore Affected Files, then evaluate any new gaps:

- 🔴 Required gaps found → set `{nn}-spec-gaps.md` Status back to OPEN, go to Step 3
- Only 🟡 Recommended or ⚪ Optional gaps → present to user; go to Step 4 on proceed, Step 3 on resolve
- No new gaps → go to Step 4

### Note on code-map reads and writes

Code-map operations happen within command steps (read at Step 2, write at Step 3.5) and do not produce their own persistent state files. On resumption:

- Step 2 re-reads the code-map fresh — fresh dedup, fresh GC of stale entries, fresh verification. Idempotent per reader contract
- Step 3.5 may re-append the same entry — duplicates are harmless: dedup at read time picks the most-recent `verified_at` and removes older rows per `../../shared/references/code-map-format.md` reader contract
- If any code-map append fails (e.g., filesystem error), re-run simply retries the append; no separate recovery needed
