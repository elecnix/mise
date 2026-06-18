---
name: mise-task-runner
description: Comprehensive guide to mise development environment tool for task running, tool/version management, and environment configuration. Use when setting up mise.toml, creating file tasks, managing tool versions across projects, or troubleshooting the mise task runner.
license: MIT
compatibility: Requires mise CLI installed (curl https://mise.run | sh). Linux, macOS, Windows supported.
metadata:
  version: "1.0"
  author: elecnix
---

# mise Development Environment Tool

mise is a unified tool and task runner that replaces asdf, direnv, and make. It manages dev tools (node, python, go, etc.), project environment variables, and task automation in one `mise.toml` config.

## Installation

```bash
curl https://mise.run | sh
eval "$(~/.local/bin/mise activate bash)"  # Add to ~/.bashrc
```

## Three Pillars

### 1. Tool Version Management
Install and activate specific tool versions per-project:

```bash
mise use node@20           # Installs node 20, adds to mise.toml
mise use -g node@lts       # Global default (adds to ~/.config/mise/config.toml)
mise install               # Installs all tools in config
mise ls                    # Lists installed versions
mise ls-remote python      # Lists available versions for a tool
```

**mise.toml `[tools]` section:**
```toml
[tools]
node = "20"          # Exact version
python = "latest"    # Latest available
go = "lts"           # Latest LTS
rust = "nightly"     # Nightly channel
# Version specifiers: 20, lts, latest, nightly, prefix:1.2, path:/custom
```

### 2. Environment Variables
Load project-specific environment variables automatically on directory change:

```bash
mise set KEY=value      # Sets config value, updates mise.toml
mise env                # Outputs shell env commands (run: eval "$(mise env)")
mise shell              # Spawns subshell with env changes
```

**mise.toml `[env]` section:**
```toml
[env]
_.file = ".env"        # Loads .env file into environment
DATABASE_URL = "postgresql://localhost/myapp"
# _.command = "op read ..."  # Load from command output
# _.template = "..."         # Template expansion
```

### 3. Task Runner
Define and run project automation with parallel dependency resolution:

```bash
mise run build           # Executes [tasks.build] or .mise/tasks/build
mise run test -- -v      # Passes args after -- to task
mise tasks               # Lists all available tasks with descriptions
mise watch build         # Runs on file changes (requires sources defined)
mise tasks edit build    # Opens $EDITOR on task file (creates if missing)
```

## Task Types

### TOML Tasks
```toml
[tasks.build]
run = "npm run build"
description = "Build the frontend"
alias = "b"              # Run as: mise run b
depends = ["lint", "test"]  # Runs before build (parallel when possible)
dir = "{{cwd}}"          # Overrides default (config root directory)
quiet = true             # Don't print "[build] $ npm run build"
```

### File Tasks (Recommended)
Executable scripts in `.mise/tasks/`, `mise-tasks/`, or `.mise/tasks/`. Better editor support (syntax highlighting, linting).

```bash
# File: .mise/tasks/test
#!/usr/bin/env bash
#MISE description="Run unit tests"
#MISE sources=["tests/**/*.rs", "src/**/*.rs"]  # For mise watch
#MISE outputs=["target/debug/test-results"]     # For caching
set -euo pipefait

cargo test
```

**File task header options:**
`#MISE description="..."` - Shows in `mise tasks`
`#MISE alias="..."` - Short name
`#MISE sources=["..."]` - Input files (for `mise watch` and caching)
`#MISE outputs=["..."]` - Output files
`#MISE depends=["..."]` - Dependencies
`#MISE dir="{{cwd}}"` - Working directory override
`#MISE quiet=true` - Hide command echo
`#MISE raw=true` - Direct stdin/stdout/stdin (blocks parallel execution)

## Task Arguments (Usage Spec)

File tasks support argument parsing via usage comments:

```bash
#!/usr/bin/env bash
#USAGE arg "<file>" help="File to process" choices=["main.rs", "lib.rs"]
#USAGE flag "--verbose" help="Enable verbose output"
#USAGE flag "-o --output <file>" help="Output file" default="out.txt"

echo "Processing ${usage_file}"
[ "${usage_verbose:-false}" = "true" ] && echo "Verbose!"
```

Run with: `mise run process main.rs --verbose`

## Environment Variables in Tasks

mise sets these for every task execution:

| Variable | Value |
|----------|-------|
| `MISE_ORIGINAL_CWD` | Directory where `mise run` was executed |
| `MISE_CONFIG_ROOT` | Directory containing `mise.toml` |
| `MISE_TASK_NAME` | Name of the task being run |
| `MISE_TASK_DIR` | Directory containing the task script |

## Task Configuration Options

Complete reference for task properties:

| Property | Type | Description |
|----------|------|-------------|
| `run` | string/array | Required. Commands to execute |
| `run_windows` | string | Windows-specific run command |
| `description` | string | Shown in `mise tasks` output |
| `alias` | string | Alternative task name |
| `depends` | array | Tasks to run before this one |
| `depends_post` | array | Tasks to run after this one completes |
| `wait_for` | array | Optional dependencies (don't add to run list) |
| `env` | object | Environment variables for this task only |
| `tools` | object | Tool versions to activate for task |
| `dir` | string | Working directory (default: config root) |
| `sources` | array | Input files for caching/watching |
| `outputs` | array | Output files for caching |
| `hide` | bool | Hide from task listings |
| `quiet` | bool | Don't print command being run |
| `silent` | bool | Hide all task output |
| `raw` | bool | Connect directly to stdin/stdout (blocks parallel) |
| `raw_args` | bool | Pass all args through without parsing |
| `interactive` | bool | Exclusive I/O lock |
| `confirm` | string/object | Prompt before running |

### Dependency Types

- `depends`: Runs before task; added to execution list
- `depends_post`: Runs after task; still added to list
- `wait_for`: Runs before if already running; NOT added to list

```toml
[tasks.lint]
run = "eslint ."

[tasks.test]
depends = ["lint"]        # Runs lint first
wait_for = ["render"]     # Waits if render is running, doesn't add to list
run = "npm test"
```

## CLI Commands

| Command | What it does |
|---------|--------------|
| `mise install [tool]` | Downloads and installs tool versions defined in config |
| `mise use <tool@version>` | Installs tool and updates config file |
| `mise use -g <tool@version>` | Sets global default (updates ~/.config/mise/config.toml) |
| `mise exec <tool> -- <cmd>` | Runs command with specific tool version in PATH |
| `mise run <task>` | Executes task (resolves dependencies, parallel executes) |
| `mise tasks` | Lists available tasks from all config sources |
| `mise watch <task>` | Runs task and re-runs when source files change |
| `mise set KEY=VALUE` | Sets config value in nearest config file |
| `mise env` | Outputs shell commands to set environment |
| `mise shell` | Spawns subshell with activated environment |
| `mise doctor` | Checks mise installation and configuration |
| `mise config` | Shows loaded config files and their paths |
| `mise current` | Shows active tool versions for current directory |
| `mise which <tool>` | Shows full path to tool executable |

## Configuration Hierarchy

Loaded in order (later overrides earlier):

1. `~/.config/mise/config.toml` - Global config
2. `~/.config/mise/conf.d/*.toml` - Global includes
3. Parent `.mise.toml` files going up directory tree
4. `./.mise.toml` or `./mise.toml` - Project config
5. `./.mise.local.toml` - Local override (not committed)

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Task not found | `mise tasks` lists available; check file executable bit |
| Tools not in PATH | Add shell activation; run `mise trust .mise.toml` |
| Task runs on every call | Ensure `sources` and `outputs` are defined correctly |
| Parallel tasks conflict | Add `raw = true` or `interactive = true` to task |
| File watch not triggering | `mise watch -v` to see what's being watched |
| Template variables empty | Use `${var:-default}` for safe defaults |
| Task runs from wrong dir | Use `dir = "{{cwd}}"` or `cd "$MISE_ORIGINAL_CWD"` |

## Template Variables

Available in task definitions and configs:

- `{{cwd}}` - Current working directory
- `{{config_root}}` - Directory with mise.toml
- `{{env.VAR}}` - Environment variable value
- `{{vars.name}}` - Variable from `[vars]` section
- `{{usage.arg_name}}` - Usage argument value

## Remote Task Includes

Share tasks across projects (experimental):

```toml
[task_config]
includes = [
  "git::https://github.com/myorg/shared-tasks.git//tasks?ref=main",
  ".mise/tasks",  # Local override for any shared tasks
]
```
