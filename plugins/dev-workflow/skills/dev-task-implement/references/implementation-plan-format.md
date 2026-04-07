# {nn}-plan.md Format

## Structure

```
# Implementation Plan — Task {nn}: {task name}

Date: {YYYY-MM-DD}
```

## Affected Files Table

```
## Affected Files

| # | File | Location | Change Type | Description |
|---|------|----------|-------------|-------------|
| 1 | `path/to/file.kt` | `ClassName.methodName` (L120-135) | modify | ... |
| 2 | `path/to/new_file.kt` | — | create | ... |

Dependencies: #2 must be completed before #1
```

## Sub-tasks (only when split was applied)

```
## Sub-tasks

| # | Sub-task | Files |
|---|----------|-------|
| A | {description} | #1, #2 |
| B | {description} | #3 |
```
