# {nn}-progress.md Format

## Structure

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

## Single-task case

When there is only one task (single task uses parent branch directly):
replace the branch row with:
`| 1 | Verify parent branch {prefix}/{id} | ⏳ Pending |`

## Status Values

- `⏳ Pending` — not yet executed
- `✅ Applied` — successfully completed
- `⏭ Skipped` — explicitly skipped by user (file changes only)
