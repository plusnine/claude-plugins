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

Invoke `bugfix:investigate` agent. Pass:
- The ticket content (summary, reproduction steps, expected/actual behavior)
- The path to the project root `CLAUDE.md`
- The path to `spec-review/source.md` under the related ticket (if available)

Write `claude-output/{ticket-id}/bugfix/investigation-report.md` with Status: DRAFT.

If the Affected Area is `unknown`:
1. Present proposed entry points to user
2. On approval: append to the project root `CLAUDE.md`'s "Investigation Entry Points" section (create the section if it does not exist) using this format:
   ```
   ### {subsystem name}
   #### {area name}
   Entry files (read in this order):
   1. `{file}` — {role}

   Fixed change targets:
   - {fixed change targets if identified, otherwise "TBD"}
   ```
   If the new area fits under an existing subsystem (H3), append under it instead of creating a new H3.
3. On rejection: proceed without registering

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

### Step 3: Invoke spec-breakdown

Execute the spec-breakdown flow for `{ticket-id}`.
spec-breakdown will detect `bugfix/investigation-report.md` (Status: FINAL) as input.

### Step 4: Invoke task-implement

For each task in `claude-output/{ticket-id}/spec-breakdown/tasks/`:
execute the task-implement flow for `{ticket-id} {nn}`.

Wait until all tasks reach completion (`{nn}-done.md` or `{nn}-skipped.md` for every `{nn}`).

### Step 5: Post-processing

After all tasks complete:

1. **CLAUDE.md update**: Propose additions to the project CLAUDE.md based on investigation findings
   (entry points, data flows, non-obvious connections discovered during investigation).
   Do not re-propose entry points already registered in Step 2.
   Present proposed changes to user → approval required. Declined proposals do not block completion.

2. **Self-improvement**: Suggest the user run `/self-improvement` for session retrospective.

3. Write `claude-output/{ticket-id}/bugfix/done.md`:
   ```
   completed: {YYYY-MM-DD}
   tasks: [{nn list}]
   claude-md-updated: {true | declined}
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
