# PR Guidelines

## Branch Naming

The parent branch (`{prefix}/{id}`) is created by the user with the appropriate prefix (e.g. `feature/`, `hotfix/`).
Task branches inherit the parent branch's prefix:

- Multiple tasks: `{prefix}/{id}_{n}` (no zero-padding, e.g. `feature/PROJ-123_1`)
- Single task: use the parent branch directly as the head branch

## Base Branch Resolution

Determines the base (merge target) branch for a task's PR:

1. From `nn - 1`, descend to find the first task with `{nn}-done.md`
2. Found → use that task's head branch (`{prefix}/{id}_{n}`) as base
3. Not found → if single task, use `develop` as base; if multiple tasks, use `{prefix}/{id}` as base
4. `{prefix}/{id}` itself is based on `develop` (default; confirm with user before creation)
5. If the resolved base branch does not exist on remote, create it (included in unified preview approval)

## Commit Guidelines

- Language: commit subject must be written in English
- Format: Semantic Commit Messages — `type: subject` (no scope)
- Allowed types: `feat`, `fix`, `refactor`, `chore`, `test`, `docs`
- 1 commit = 1 type — if multiple types apply, split into separate commits
- Target ~100 lines per commit; strongly recommend splitting at >200 lines
- Grouping is based on actual `git diff` at commit time, not the implementation plan

## PR Content

- Always created as Draft PR
- Use the repository's PR template if present (search `.github/pull_request_template.md`)
- If no template exists, use `pr-default-template.md` (translate section headers per `language` in `~/.claude/settings.json`; fallback: English)
- PR title: `[{id}] {change summary}` — no commit prefix, no task count
- Ticket URL and specification URL: extract from `claude-output/{id}/spec-review/source.md`

### Screenshot Rules

Determine screenshot needs based on the nature of changes:

- **Not needed** (no UI change): state that screenshots are not needed. No placeholder.
- **Needed, no dark mode**: single `<img src="" width="320" />` placeholder
- **Needed, dark mode supported**: separate `## Light` / `## Dark` subsections
- **Before/After**: use table within each section

Dark mode support is determined by checking the repository for dark mode resources or theme configuration.
See `pr-default-template.md` for markup examples.

### Existing PR Handling

Before creating a new PR, check for existing PRs on the same head branch via `gh pr list --head {branch}`:

- **Exists**: preview diff between current PR body and proposed content → confirm update via `gh pr edit`
- **Does not exist**: create new PR
- User may request adjustments via prompt before final approval

## Unified Preview Format

Present commits and PR content together for approval:

```
🔷 Commits
━━━━━━━━━━━━━━━━━━━━━━
[1] type: subject
    - file1
    - file2

[2] type: subject
    - file3

🟢 PR (new) / 🟡 PR (update)
━━━━━━━━━━━━━━━━━━━━━━
Base: {prefix}/{id} (existing) / {prefix}/{id} (new, from develop)
Title: [{id}] change summary
Body:
(PR body content)
```

- Numbered commits allow per-commit modification instructions (e.g. "[1] change message to ...")
- Single approval executes: commit → push → PR create/update
- User may request separation of push and PR creation

## Sensitive Content

Before PR creation, check repository visibility via `gh repo view --json isPrivate`:
- Private repo → internal URLs (JIRA, Confluence, etc.) are exempt from sensitive content warnings
- Public repo → display sensitive content warning at preview and require explicit approval
