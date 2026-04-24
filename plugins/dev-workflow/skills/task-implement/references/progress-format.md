# {nn}-progress.md Format

## Structure

The branch row at position 1 takes one of two shapes depending on whether the current `{id}` was decomposed into multiple tasks. The decision is owned by `pr-guidelines.md` "Branch Naming"; this format file declares the resulting shapes only.

### Multi-task case (more than one task in `tasks/`)

```
# Progress — Task {nn}: {task name}

| # | Item | Status |
|---|------|--------|
| 1 | Create branch {prefix}/{id}_{n} | ⏳ Pending |
| 2 | `path/to/file` | ⏳ Pending |
| 3 | `path/to/new_file` | ⏳ Pending |
| ... | ... | ... |
| N | Commit | ⏳ Pending |
| N+1 | Push | ⏳ Pending |
| N+2 | Create draft PR | ⏳ Pending |
```

### Single-task case (exactly one task in `tasks/`, using parent branch directly)

```
# Progress — Task {nn}: {task name}

| # | Item | Status |
|---|------|--------|
| 1 | Verify parent branch {prefix}/{id} | ⏳ Pending |
| 2 | `path/to/file` | ⏳ Pending |
| 3 | `path/to/new_file` | ⏳ Pending |
| ... | ... | ... |
| N | Commit | ⏳ Pending |
| N+1 | Push | ⏳ Pending |
| N+2 | Create draft PR | ⏳ Pending |
```

## Status Values

- `⏳ Pending` — not yet executed
- `✅ Applied` — successfully completed
- `⏭ Skipped` — explicitly skipped by user (file changes only)
