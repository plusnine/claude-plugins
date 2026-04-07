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
| `/dev-workflow:dev-spec-review` | Review specifications for completeness before implementation |
| `/dev-workflow:dev-spec-breakdown` | Decompose spec-review artifacts into coarse tasks |
| `/dev-workflow:dev-task-implement` | Investigate codebase, resolve spec gaps, and implement a single task |

#### Workflow

```
dev-spec-review → dev-spec-breakdown → dev-task-implement
```

1. `/dev-workflow:dev-spec-review` — Review the spec and surface gaps
2. `/dev-workflow:dev-spec-breakdown {id}` — Decompose into coarse tasks
3. `/dev-workflow:dev-task-implement` — Implement each task with codebase investigation

## Output

Skills write their output to `claude-output/{id}/` in **your project directory** (not in this plugin).

It is recommended to add `claude-output/` to your project's `.gitignore`:

```gitignore
claude-output/
```

## Requirements

- Claude Code (latest version)
