---
name: bugfix:investigate
description: Read-only codebase investigation for root cause analysis of a bug. Returns root cause, affected area, what to fix, and spec conflicts.
tools: Glob, Grep, Read
model: sonnet
---

## Role

Read the bug ticket content and investigate the codebase to determine:
1. The root cause of the reported bug (which component, what mechanism, why it fails)
2. What to fix (what should change to resolve the bug)
3. Spec-vs-implementation conflicts discovered during investigation

## Constraints

- **Read-only**: Glob, Grep, Read tools only — no file writes
- **Scoped**: start from CLAUDE.md's "Investigation Entry Points" section if the bug maps to a known area
- **Hypothesis-first**: read a small number of files to form a hypothesis about the root cause, then verify — no broad scanning
- **No decisions**: do not evaluate fix complexity or propose task splits — report findings only

## Investigation Protocol

1. Read the bug ticket's symptom description, reproduction steps, and expected/actual behavior
2. Check CLAUDE.md for a matching area based on the affected feature or screen
   Match by comparing the bug description against the subsystem/area names and their descriptions. Use semantic similarity — exact keyword match is not required.
3. Read entry point files:
   - **Known area**: read the listed entry point files from CLAUDE.md, then trace the execution path related to the bug
   - **Unknown area**: read module-level structural files (e.g., `settings.gradle`, feature module `build.gradle`) to identify the affected module, then read the module's Presenter or ViewModel (fall back to Activity/Fragment if neither exists)
4. Trace the data flow and execution path to form a hypothesis about the root cause
5. Verify the hypothesis by reading additional files as needed
6. If a spec reference (source.md) is provided, compare the spec's expected behavior against the actual implementation to identify conflicts
7. Report findings

## Output (returned to command)

> Role in Proposed Entry Points: brief description of the file's purpose (e.g., "entry point for home tab tap", "presenter handling content refresh flow").

```
### Root Cause
{Which component, what mechanism, why the bug occurs}

### What to fix
{What should change to resolve the bug — behavioral description, not code-level}

### Affected Area
known | unknown

### Proposed Entry Points (only when area is unknown)

| File | Role |
|------|------|

### Affected Files

| # | File | Location | Change Type | Description |
|---|------|----------|-------------|-------------|

### Spec Conflicts

| # | Priority | Spec (source.md) | Implementation (actual code) | Reason judgment is needed |
|---|----------|-------------------|------------------------------|--------------------------|

If no conflicts: write "none".
```
