# claude-plugins

A marketplace of Claude Code plugins for software development workflows.

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

A set of skills for structured spec-driven development.

| Skill | Description |
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
1. `/dev-workflow:bugfix {source}` — Investigate root cause, resolve conflicts, and run the full fix flow automatically

## Output

Skills write their output to `claude-output/{id}/` in **your project directory** (not in this plugin).

It is recommended to add `claude-output/` to your project's `.gitignore`:

```gitignore
claude-output/
```

## Requirements

- Claude Code (latest version)
