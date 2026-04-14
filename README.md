# claude-plugins

A marketplace of Claude Code plugins for software development workflows.

## Design approach

The plugin provides commands for spec-driven development and bug-fix workflows, each structured around explicit checkpoints. Every command:

- requires **user approval** at consequential decisions — plan review, per-file changes, commit grouping, PR creation
- persists state to `claude-output/{id}/` files so interrupted sessions can be resumed
- blocks forward progress when 🔴 Required spec gaps or spec conflicts remain unresolved
- delegates code investigation to **read-only agents** separated from code-modifying orchestration

The plugin's core does not encode domain-specific conventions (build systems, architecture patterns, file extensions). Projects can supply these via the root `CLAUDE.md` (see [CLAUDE.md Integration](#claudemd-integration)); absent that, agents fall back to general codebase heuristics and propose findings for user approval.

## Install

Add this marketplace to Claude Code:

```shell
/plugin marketplace add plusnine/claude-plugins
```

Then install the plugin:

```shell
/plugin install dev-workflow@plusnine-claude-plugins
```

## Plugins

### dev-workflow

A set of commands for structured spec-driven development.

| Command | Description |
|-------|-------------|
| `/dev-workflow:spec-review` | Review specifications for completeness before implementation |
| `/dev-workflow:spec-breakdown` | Decompose spec-review artifacts into coarse tasks |
| `/dev-workflow:task-implement` | Investigate codebase, resolve spec gaps, implement a single task, and create a draft PR |
| `/dev-workflow:bugfix` | Investigate a bug from a BTS/ITS ticket, resolve spec conflicts, and orchestrate the full fix flow |

#### Workflow

```
# Feature development
spec-review → spec-breakdown → task-implement

# Bug fix
bugfix → (spec-breakdown → task-implement)
```

**Feature development:**
1. `/dev-workflow:spec-review {source}` — Review the spec and surface gaps
2. `/dev-workflow:spec-breakdown {id}` — Decompose into coarse tasks
3. `/dev-workflow:task-implement {id} {nn}` — Implement a task and create a draft PR

**Bug fix:**
1. `/dev-workflow:bugfix {source}` — Investigate root cause, resolve conflicts, and orchestrate spec-breakdown and task-implement for the full fix flow (each step requires user approval)

## Components

| Command | Invokes | Reads | Writes |
|---|---|---|---|
| `/dev-workflow:spec-review {source}` | — | `{source}` (URL or file path) | `claude-output/{id}/spec-review/{source.md, review.md}` |
| `/dev-workflow:spec-breakdown {id}` | — | `spec-review/source.md` or `bugfix/investigation-report.md` (FINAL) | `claude-output/{id}/spec-breakdown/{plan.md, tasks/*.md}` |
| `/dev-workflow:task-implement {id} {nn}` | `task-implement:investigate` agent | `spec-breakdown/tasks/{nn}-*.md`, `spec-review/source.md` or `bugfix/investigation-report.md` | `claude-output/{id}/task-implement/{nn}-{plan,progress,spec-gaps,done,skipped}.md` + branch/commit/draft PR |
| `/dev-workflow:bugfix {source}` | `bugfix:investigate` agent, `/dev-workflow:spec-breakdown`, `/dev-workflow:task-implement` | ticket URL | `claude-output/{id}/bugfix/{meta,investigation-report,spec-conflicts,done}.md` |

```mermaid
flowchart LR
    SR["/dev-workflow:spec-review"]
    SB["/dev-workflow:spec-breakdown"]
    TI["/dev-workflow:task-implement"]
    BF["/dev-workflow:bugfix"]
    TIA(["task-implement:investigate"])
    BFA(["bugfix:investigate"])

    SR -.-> SB
    SB -.-> TI
    TI --> TIA
    BF --> BFA
    BF --> SB
    BF --> TI

    classDef agent fill:#eef,stroke:#99f
    class TIA,BFA agent
```

Legend: solid arrow = programmatic invocation; dashed arrow = typical user sequence.

Agents (`task-implement:investigate`, `bugfix:investigate`) are read-only: they use Glob/Grep/Read tools only and cannot modify files.

## Behavior

### Resumability

Each command is resumable. On re-run with the same `{id}`, resume position is derived from the files present in `claude-output/{id}/`. See the Checkpoint section in each command file for exact resume rules.

### CLAUDE.md Integration

The investigate agents read the project root `CLAUDE.md` for codebase context. The plugin does not modify `CLAUDE.md` or require any specific section structure.

### Language

File output is always written in English. User-facing messages (previews, prompts, diffs) follow the `language` setting in `~/.claude/settings.json`, falling back to English if unset.

### Boundaries

What this plugin does not do:

- Auto-merge PRs — all PRs are created as Draft
- Proceed past unresolved 🔴 Required spec gaps or spec conflicts
- Decide scope (task splits, file edits, branch creation, commits) without user approval
- Modify the project root `CLAUDE.md`

## Output

Skills write their output to `claude-output/{id}/` in **your project directory** (not in this plugin).

```
claude-output/{id}/
├── spec-review/
│   ├── source.md          # cached spec content
│   └── review.md          # review result
├── spec-breakdown/
│   ├── plan.md            # coarse task list
│   └── tasks/
│       ├── 01-*.md        # task prompt files
│       └── 02-*.md
├── task-implement/
│   ├── {nn}-plan.md
│   ├── {nn}-progress.md
│   ├── {nn}-spec-gaps.md
│   └── {nn}-done.md       # or {nn}-skipped.md
└── bugfix/
    ├── meta.md
    ├── investigation-report.md
    ├── spec-conflicts.md
    └── done.md
```

It is recommended to add `claude-output/` to your project's `.gitignore`:

```gitignore
claude-output/
```

## Requirements

- Claude Code (latest version)
