---
description: Investigate a bug from BTS/ITS ticket, resolve spec conflicts, and orchestrate spec-breakdown and task-implement for fix implementation.
argument-hint: "{source}"
---

All file output must be written in English regardless of content origin.
User-facing output (messages, previews, etc.) must be in the language specified by `language` in `~/.claude/settings.json` (fallback: English).

## Invocation

`/bugfix {source}`

Where `{source}` is a URL (Jira, Linear, etc.) to the bug ticket.

## Process

### Step 0: Fetch ticket

Fetch the ticket content from `{source}`.
Extract the ticket ID from the URL to use as `{ticket-id}` for output paths.
Present a summary (symptom, expected behavior, actual behavior, reproduction steps) to the user.
Wait for user approval before proceeding.

### Step 0b: Resolve related ticket

Ask the user for the related feature ticket ID (the original feature/spec this bug relates to).
- If provided: check `claude-output/{related-ticket-id}/spec-review/source.md` exists
  - Exists → use as spec reference in Step 2
  - Does not exist → inform user, proceed without spec reference
- If none: proceed without spec reference

### Step 1: Create branch

Ask the user for the base branch to create the bugfix branch from.
Create working branch: `bugfix/{ticket-id}` from the specified base branch.
Write `claude-output/{ticket-id}/bugfix/meta.md`:

```
ticket-id: {ticket-id}
related-ticket-id: {related-ticket-id or "none"}
base-branch: {selected release branch}
branch-prefix: bugfix
```

Present branch name to user → **Approval ①**.

### Step 2: Investigate

Before invoking the agent, load Index Hints from code-map per `../../shared/references/code-map-format.md`:

1. Resolve `{repo-name}` per code-map-format.md (`basename $(git rev-parse --show-toplevel)` lowercased). If not in a git repo: skip the hints path entirely
2. If `claude-output/_index/{repo-name}/code-map.md` exists:
   a. Extract keywords from the ticket (ticket title + prominent noun phrases from symptom/expected/actual). Expand with semantic equivalents — synonyms, related terms, alternative namings the codebase may use (query expansion). Leverage CLAUDE.md for domain vocabulary
   b. Filter entries by case-insensitive substring match on `concept` column against any keyword (base or expanded)
   c. Deduplicate: for each duplicated concept, keep the entry with the most recent `verified_at` (by git commit recency). Tiebreaker for same verified_at: keep the last-written (bottom-most) row. Remove older duplicates from the file
   d. For each remaining candidate entry, verify:
      - All paths in `starting_points` exist (remove entry from file if any path missing)
      - `git diff {verified_at}..HEAD -- {starting_points}`: success + no diff → high confidence; success + diff → lower confidence; command failure (verified_at unreachable) → remove entry from file
   e. Pass verified entries to the agent as Index Hints (markdown table format, see code-map-format.md "Agent integration")
3. If code-map does not exist or no hints survived: skip the hints path

Surface to user a one-line summary after the read phase: `Index: N hints used (M high, K lower), S stale removed`. If no hints used, `Index: no matches`.

Append a metrics line to `claude-output/_index/{repo-name}/code-map-metrics.log` per `../../shared/references/code-map-format.md` Metrics Log section:
`{ISO8601-UTC}<TAB>read<TAB>matched:{N},verified:{M},removed:{S},passed:{K}`

Invoke `bugfix:investigate` agent. Pass:
- The ticket content (summary, reproduction steps, expected/actual behavior)
- The path to the project root `CLAUDE.md`
- The path to `spec-review/source.md` under the related ticket (if available)
- Index Hints (if any)

Write `claude-output/{ticket-id}/bugfix/investigation-report.md` with Status: DRAFT.

Present root cause and what to fix to user.

### Step 2b: Conflict resolution loop

If the investigation report contains spec-vs-implementation conflicts:

If only 🟡 Recommended or ⚪ Optional conflicts exist (no 🔴 Required):
write `claude-output/{ticket-id}/bugfix/spec-conflicts.md` with Status: RESOLVED and present to user. Skip to Step 2c.

If 🔴 Required conflicts exist:
1. Write `claude-output/{ticket-id}/bugfix/spec-conflicts.md` with Status: OPEN
2. Present conflicts to user. For each conflict, user decides:
   - (a) Spec is correct → fix implementation to match spec
   - (b) Implementation is correct → note spec update needed (out of bugfix scope)
   - (c) Both incorrect → user defines correct behavior
3. Record resolutions in `spec-conflicts.md` Resolution Log
4. If resolution requires verifying new code areas: re-invoke `bugfix:investigate` agent, passing the updated conflict resolutions alongside the original ticket content
5. Repeat until no 🔴 Required conflicts remain
6. Update `spec-conflicts.md` Status to RESOLVED

If no conflicts: skip this step.

### Step 2c: Finalize investigation report

Update `investigation-report.md`:
- Incorporate conflict resolution decisions into "What to fix" section
- Set Status to FINAL

Present finalized report to user → **Approval ②**.

After Approval ②, append a new entry to `claude-output/_index/{repo-name}/code-map.md` per `../../shared/references/code-map-format.md`. Resolve `{repo-name}` fresh per code-map-format.md (`basename $(git rev-parse --show-toplevel)` lowercased; skip this entire step if not in a git repo):

- `concept`: ticket summary/title (e.g., Jira "summary" field, Linear "title" field — the human-readable description, not the ticket ID), normalized (lowercase, kebab/snake → space-separated, trimmed, drop trailing period if any). Example: Jira "BUG-123: Password reset link broken" → `"password reset link broken"`
- `starting_points`: agent's reported Starting Points from investigation-report.md (pipe-joined, priority order preserved)
- `verified_at`: current `git rev-parse --short HEAD`

Create `claude-output/_index/{repo-name}/` if it does not exist. Skip if the agent returned no Starting Points.

Surface to user after append: `Index: appended '{concept}' → {starting_points} @ {verified_at}`. If skipped, state why (e.g., `Index: skipped — not a git repo` / `Index: skipped — no starting points`).

Append a metrics line to `claude-output/_index/{repo-name}/code-map-metrics.log`:
`{ISO8601-UTC}<TAB>write<TAB>appended:{1 if appended, 0 if skipped}`

### Step 3: Invoke spec-breakdown

Execute the spec-breakdown flow for `{ticket-id}`.
spec-breakdown will detect `bugfix/investigation-report.md` (Status: FINAL) as input.

### Step 4: Invoke task-implement

For each task in `claude-output/{ticket-id}/spec-breakdown/tasks/`:
execute the task-implement flow for `{ticket-id} {nn}`.

Wait until all tasks reach completion (`{nn}-done.md` or `{nn}-skipped.md` for every `{nn}`).

### Step 5: Post-processing

After all tasks complete:

1. **Self-improvement**: Suggest the user run `/self-improvement` for session retrospective.

2. Write `claude-output/{ticket-id}/bugfix/done.md`:
   ```
   completed: {YYYY-MM-DD}
   tasks: [{nn list}]
   ```

## Checkpoint

On re-run, resume position is determined by existing files:

| State | Resume from |
|-------|-------------|
| `bugfix/done.md` exists | Already complete |
| All `task-implement/{nn}-done.md` or `{nn}-skipped.md` exist | Step 5 (post-processing) |
| `task-implement/{nn}-progress.md` or `{nn}-plan.md` exists | Step 4 (task-implement internal checkpoint) |
| `spec-breakdown/tasks/` has files | Step 4 (invoke task-implement) |
| `bugfix/investigation-report.md` Status: FINAL | Step 3 (invoke spec-breakdown) |
| `bugfix/spec-conflicts.md` Status: RESOLVED & `investigation-report.md` Status: DRAFT | Step 2c (finalize report) |
| `bugfix/spec-conflicts.md` Status: OPEN | Step 2b (conflict resolution loop) |
| `bugfix/investigation-report.md` Status: DRAFT & no `spec-conflicts.md` | Step 2b (check for conflicts) |
| `bugfix/meta.md` exists & no `investigation-report.md` | Step 2 (investigate) |
| No `bugfix/meta.md` | Step 0 (fetch ticket) |
