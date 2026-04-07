---
description: Orchestrates investigation, gap resolution, sub-task evaluation, and implementation for a single task. Owns all file writes.
argument-hint: "{id} {nn}"
---

## Invocation

`/task-implement {id} {nn}`

Future extension: `/task-implement {id} all`
— iterate over `claude-output/{id}/spec-breakdown/tasks/` in filename order, run each task sequentially.

## Process

### Step 1: Load task

Read `claude-output/{id}/spec-breakdown/tasks/{nn}-*.md`.

### Step 2: Investigate

Invoke `task-implement:investigate` agent. Pass:
- The full content of the task prompt file
- The path to the project root `CLAUDE.md` (the one containing "Investigation Entry Points"; agent will read it directly)

If the Affected Files list is empty: present the findings to the user and ask how to proceed. Do not continue to Step 2b.

### Step 2b: Register new entry points (unknown area only)

If the investigate agent reports `Area: unknown`:
1. Present proposed entry points to user
2. On approval: append to the project root `CLAUDE.md`'s "Investigation Entry Points" section using this format:
   ```
   ### {subsystem name} (e.g. "リスト面系", "記事面系", or a new name if the area is distinct)
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

### Step 5: Write plan and apply

Write `claude-output/{id}/task-implement/{nn}-plan.md`.
Present the plan to the user. Wait for explicit approval before proceeding.
On approval: Write `claude-output/{id}/task-implement/{nn}-progress.md` listing all files from the plan as ⏳ Pending.
For each file change (apply in dependency order if Dependencies are specified in the plan):
1. Apply the change — requires user approval per the Write Safety Gate
2. On approval: update `{nn}-progress.md` → ✅ Applied
3. On rejection: update `{nn}-progress.md` → ❌ Rejected, continue to next file

### Step 6: Mark as done

If ❌ Rejected entries exist: present them to the user before asking for completion.
Ask the user: "Mark this task as done?"
On explicit approval: rename `{nn}-progress.md` → `{nn}-done.md`.

## Checkpoint

On re-run, resume position is determined by existing files:

| State | Resume from |
|-------|-------------|
| `{nn}-done.md` exists | Already complete |
| `{nn}-progress.md` exists & no ⏳ Pending remain | Step 6 (mark as done) |
| `{nn}-progress.md` exists & ⏳ Pending remain | Step 5 (apply remaining ⏳ Pending files) |
| `{nn}-plan.md` exists | Step 5 (present plan for approval, write progress.md, start applying) |
| `{nn}-spec-gaps.md` exists & Status: OPEN | Step 3 (gap resolution loop) |
| `{nn}-spec-gaps.md` exists & Status: RESOLVED | Step 2 (re-investigate to restore Affected Files; if Area: unknown, also run Step 2b), then Step 4 |
| Otherwise | Step 2 (investigate) |
