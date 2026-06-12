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

### Resolve target repository (precondition)

If `claude-output/{ticket-id}/meta.md` does not exist, run the detection algorithm in `../../shared/references/meta-format.md` ("Detection algorithm") and write meta.md.

If meta.md exists, validate it per `../../shared/references/meta-format.md` ("Validation"). On validation failure, exit with an error in the user's configured language and require manual correction before re-run.

This section runs between Step 0b and Step 1. All subsequent git/gh/glab operations follow the contract in `meta-format.md` "Reading".

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

Before invoking the agent, load Index Hints from code-map per `../../shared/references/code-map-format.md` (sections: `{repo-name}` resolution, Read policy, Agent integration). Skip if not in a git repo.

Surface to user after the read phase: `Index: N hints used (M high, K lower), S stale removed`. If no hints used, `Index: no matches`.

Append a metrics line per `code-map-format.md` Metrics Log section (read trigger).

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

After Approval ②, save the investigate agent's reply (the most recent invocation, including any retry from Step 2b) verbatim to `claude-output/{ticket-id}/bugfix/_agent-reply.md`, then invoke the `code-map-write` skill with the following args:

```
target-repository-path: <value of target-repository-path from claude-output/{ticket-id}/meta.md>
agent-reply-source: claude-output/{ticket-id}/bugfix/_agent-reply.md
ref: {ticket-id}
task: none
```

The skill executes the full write pipeline (v1 legacy migration, 6-layer validation with auto-fix, append, metrics logging) per `../../shared/references/code-map-format.md` and returns a single-line surface message. Display the message to the user verbatim.

**Retry orchestration** (command-side, kept here because it requires re-invoking the investigate agent): if the returned message starts with `Index: write rejected`, perform exactly one retry per `../../shared/references/code-map-format.md` "Retry protocol":
1. Re-invoke the `bugfix:investigate` agent with the original Step 2 input plus the retry suffix (substituting the failing layer, error message, and offending substring from the rejection).
2. Save the retry reply to `claude-output/{ticket-id}/bugfix/_agent-reply.retry.md`.
3. Re-invoke the `code-map-write` skill with `agent-reply-source: claude-output/{ticket-id}/bugfix/_agent-reply.retry.md`. Display the returned message to the user.

Do not retry a third time. Downstream steps (Step 3 onward) proceed regardless of the code-map write outcome.

### Step 3: Invoke spec-breakdown

Read `../spec-breakdown/command.md` first to load the full process, then execute its complete flow for `{ticket-id}`.
spec-breakdown will detect `bugfix/investigation-report.md` (Status: FINAL) as input.
Phase 1 (plan.md approval) is mandatory — do not generate `tasks/` files until the user approves `plan.md`.

### Step 4: Invoke task-implement

For each task in `claude-output/{ticket-id}/spec-breakdown/tasks/`:
1. Read `../task-implement/command.md` first to load the full process.
2. Execute its complete flow (Steps 0–6) for `{ticket-id} {nn}`.
3. `{nn}-done.md` is **auto-created by task-implement Step 6 via rename from `{nn}-progress.md`** — never write `{nn}-done.md` manually.

Wait until all tasks reach completion (`{nn}-done.md` or `{nn}-skipped.md` for every `{nn}`).

**Invariant — implementation always goes through task-implement**:
All code changes and PR creation must be performed by invoking task-implement, without exception.
This applies even when earlier steps (spec-breakdown, investigation, branch creation) are skipped
due to re-fix runs or checkpoint resumption. Implementing code or creating PRs directly from this
command — bypassing task-implement — is not permitted regardless of how simple the change appears.

### Step 5: Post-processing

After all tasks complete:

1. **Self-improvement**: Suggest the user run `/self-improvement` for session retrospective.

2. Write `claude-output/{ticket-id}/bugfix/done.md`:
   ```
   completed: {YYYY-MM-DD}
   tasks: [{nn list}]
   ```

## Checkpoint

On re-run, resume position is determined by existing files.

**Precondition**: any resume position below assumes `claude-output/{ticket-id}/meta.md` exists per "Resolve target repository". If `{ticket-id}` has been resolved but `meta.md` is missing, run "Resolve target repository" before resuming from any other state.

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

### Note on code-map reads and writes

Code-map operations happen within command steps (read at Step 2, write at Step 2c via the `code-map-write` skill). The only persistent state files produced are the transient `claude-output/{ticket-id}/bugfix/_agent-reply.md` and `_agent-reply.retry.md` written as skill inputs (safely overwritable across resumptions). On resumption:

- Step 2 re-reads the code-map fresh — fresh dedup, fresh GC of stale entries, fresh verification. Idempotent per reader contract
- Step 2c may re-invoke the skill with the same input — duplicates in the appended JSONL are harmless: dedup at read time picks the most-recent `verified_at` and removes older rows per `../../shared/references/code-map-format.md` reader contract
- If any code-map append fails (e.g., filesystem error), re-run simply re-invokes the skill; no separate recovery needed
