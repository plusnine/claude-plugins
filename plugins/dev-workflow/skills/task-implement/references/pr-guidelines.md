# PR Guidelines

## Branch Naming

The parent branch (`{prefix}/{id}`) is created by the user with the appropriate prefix (e.g. `feature/`, `hotfix/`).
Task branches inherit the parent branch's prefix:

- Multiple tasks: `{prefix}/{id}_{n}` (no zero-padding, e.g. `feature/PROJ-123_1`)
- Single task: use the parent branch directly as the head branch

### Bugfix flow

The parent branch is created by the bugfix command with prefix `bugfix/`:
- Multiple tasks: `bugfix/{ticket-id}_{n}`
- Single task: use `bugfix/{ticket-id}` directly as the head branch

## Base Branch Resolution

Determines the base (merge target) branch for a task's PR:

1. From `nn - 1`, descend to find the first task with `{nn}-done.md`
2. Found → use that task's head branch (`{prefix}/{id}_{n}`) as base
3. Not found → if multiple tasks, use `{prefix}/{id}` as base
4. Not found → use `bugfix/meta.md` `base-branch` if exists, else ask the user which branch to use as base
5. `{prefix}/{id}` creation: from `bugfix/meta.md` `base-branch` if exists, else ask the user which branch to use (confirm with user before creation)
6. If the resolved base branch does not exist on remote, create it (included in unified preview approval)

## Commit Guidelines

- Language: commit subject must be written in English
- Format: Semantic Commit Messages — `type: subject` (no scope)
- Allowed types: `feat`, `fix`, `refactor`, `chore`, `test`, `docs`
- 1 commit = 1 type — if multiple types apply, split into separate commits
- Target ~100 lines per commit; strongly recommend splitting at >200 lines
- Grouping is based on actual `git diff` at commit time, not the implementation plan

## PR/MR Content

> Terminology: "PR" (Pull Request) and "MR" (Merge Request) refer to the same concept. GitHub and Bitbucket use "PR"; GitLab uses "MR". This document uses "PR/MR" where platform-neutral phrasing is needed, and "PR" where the sentence already covers both.

- Always created as Draft PR/MR
- Use the repository's PR/MR template if present. Search per platform's official spec:
  - **GitHub**: a single template at `pull_request_template.md` / `PULL_REQUEST_TEMPLATE.md` (case-insensitive) in the repo root, `docs/`, or `.github/`; or multiple templates as `*.md` under a `PULL_REQUEST_TEMPLATE/` subdirectory in any of those three locations
  - **GitLab**: `.gitlab/merge_request_templates/*.md` on the default branch
  - **Bitbucket Cloud**: no file-based spec — the default description is stored in repository settings (Pull requests → Default description) and is not retrievable from the repo tree. Skip file search and fall back to the default template
- If no template exists, use `pr-default-template.md` (translate section headers per `language` in `~/.claude/settings.json`; fallback: English)
- PR title: `[{id}] {change summary}` — no commit prefix, no task count
- Ticket URL and specification URL: extract from `claude-output/{id}/spec-review/source.md` or `claude-output/{id}/bugfix/meta.md`

### Content Restrictions

Applies to PR body, commit messages, and PR/code review comments.

- **No AI-workflow / internal tooling references**: must not disclose the AI-assisted workflow structure used to produce the change. Forbidden elements:
  - File paths under `claude-output/` or any other workflow-managed working directory
  - Names of skills, agents, commands, or plugins (e.g. `dev-workflow`, `spec-breakdown`, `bugfix:investigate`, `task-implement`)
  - Filenames of workflow artifacts (e.g. `investigation-report.md`, `plan.md`, `progress.md`, `spec-gaps.md`, `meta.md`)
  - Internal task / phase numbering originating from the workflow (e.g. "Task 01", "sub-task A", "Phase 2", "Approval ②")
  - Workflow-specific section titles when used verbatim as headings (e.g. "Affected Files", "Starting Points")
  - **Allowed**: `Co-Authored-By: Claude ...` commit trailer (explicit attribution, expected). Also allowed: common engineering vocabulary that happens to be used in the workflow but is standard industry terminology (e.g. "root cause", "impact", "regression", "affected modules" as prose).
  - If investigation context is useful for reviewers, inline the findings as prose — do not link to workflow artifacts.

- **No negative-scope disclaimers**: do not list what the PR did *not* do. Do not write a Notes / 備考 section whose only content is "related ticket X is handled separately" or "Y is intentionally left unchanged". If it is not in the diff, the diff already says so. Omit the section entirely.
  - Exception: if the PR *preserves* something in a non-obvious way (e.g. intentionally keeping a behavior that looks like a bug), that is worth stating.

- **Inapplicable sections**: when a template section does not apply, keep the heading and write an explicit not-applicable marker in the body (e.g. `なし` / `None`). Do not leave the body blank, and do not write `N/A`.
  - Rationale: blank bodies can be read as "forgot to fill in"; `N/A` is a locale-mismatched artifact in non-English templates. An explicit marker signals "checked, not applicable".

### Screenshot Rules

Determine screenshot needs based on the nature of changes:

- **Not needed** (no UI change): state that screenshots are not needed. No placeholder.
- **Needed, no dark mode**: single `<img src="" width="320" />` placeholder
- **Needed, dark mode supported**: separate `## Light` / `## Dark` subsections
- **Before/After**: use table within each section

Dark mode support is determined by checking the repository for dark mode resources or theme configuration.
See `pr-default-template.md` for markup examples.

### Existing PR/MR Handling

Before creating a new PR/MR, check for existing ones on the same head branch using the platform's hosting CLI or REST API (e.g. `gh pr list --head {branch}` for GitHub, `glab mr list --source-branch {branch}` for GitLab, Bitbucket REST API for Bitbucket):

- **Exists**: preview diff between current PR/MR body and proposed content → confirm update via the platform's edit command (e.g. `gh pr edit`, `glab mr update`, or Bitbucket REST API)
- **Does not exist**: create new PR/MR
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

🟢 PR/MR (new) / 🟡 PR/MR (update)
━━━━━━━━━━━━━━━━━━━━━━
Base: {prefix}/{id} (existing) / {prefix}/{id} (new, from {base-branch})
Title: [{id}] change summary
Body:
(PR body content)
```

- Numbered commits allow per-commit modification instructions (e.g. "[1] change message to ...")
- Single approval executes: commit → push → PR create/update
- User may request separation of push and PR creation

## Sensitive Content

Before PR/MR creation, check repository visibility using the platform's hosting CLI or REST API (e.g. `gh repo view --json isPrivate` for GitHub, `glab repo view` for GitLab, Bitbucket REST API for Bitbucket):
- Private repo → internal URLs (JIRA, Confluence, etc.) are exempt from sensitive content warnings
- Public repo → display sensitive content warning at preview and require explicit approval
