---
name: mise-task-runner
description: Orchestrate multi-step project workflows using mise task definitions with dependency management, argument handling, and file tracking. Use when defining, running, or debugging tasks; managing build/test/deploy pipelines; creating file tasks with arguments; or setting up task caching and watch mode.
license: MIT
compatibility: Requires mise CLI installed (curl https://mise.run | sh). Works on Linux, macOS, and Windows.
metadata:
  version: "1.0"
author: elecnix
---

# mise Tasks Orchestration

Orchestrate project workflows using mise `[tasks]` section with dependency management, argument handling, and file tracking.

## When to Use This Skill

**Explicit triggers**:
- User mentions `mise tasks`, `mise run`, `[tasks]` section
- User needs task dependencies: `depends`, `depends_post`, `wait_for`
- User wants workflow automation in `.mise.toml`
- User mentions task arguments or `usage` spec

**Prescriptive triggers**:
- Multi-step workflows detected (test suites, build pipelines, migrations)

## Quick Reference

### Task Definition

```toml
[tasks.build]
description = "Build the project"
run = "cargo build --release"
```

### Running Tasks

```bash
mise run build          # Run single task
mise run test build     # Run sequentially
mise run 'test ::: build' # Run in parallel
mise r build            # Short form
```

### Dependency Types

| Type | Syntax | When |
|------|--------|------|
| `depends` | `depends = ["lint"]` | Run BEFORE task |
| `depends_post` | `depends_post = ["notify"]` | Run AFTER success |
| `wait_for` | `wait_for = ["db"]` | Wait if already running |

### Key Task Properties

| Property | Purpose | Example |
|----------|---------|---------|
| `run` | Required: command(s) | `"cargo build"` |
| `description` | AI discoverability | `"Run tests, exit 1 on failure"` |
| `alias` | Short name | `alias = "b"` |
| `dir` | Working directory | `dir = "packages/frontend"` |
| `depends` | Dependencies | `depends = ["build"]` |
| `env` | Task-specific env | `env = { LOG_LEVEL = "debug" }` |
| `hide` | Hidden tasks | `hide = true` |
| `sources` | Input files for caching | `sources = ["src/**/*.rs"]` |
| `outputs` | Output files for caching | `outputs = ["target/release/myapp"]` |
| `confirm` | Prompt before run | `confirm = "Proceed?"` |
| `quiet` | Suppress command echo | `quiet = true` |
| `raw` | Direct I/O (no parallel) | `raw = true` |
| `tools` | Task-specific versions | `tools = { python = "3.9" }` |

## File Tasks (Recommended)

Executable scripts in `.mise/tasks/`:

```bash
# File: .mise/tasks/test
#!/usr/bin/env bash
#MISE description="Run tests with coverage"
#MISE sources=["tests/**/*.rs", "src/**/*.rs"]
#MISE outputs=["target/debug/coverage.xml"]
set -euo pipefail

cargo test --coverage
```

## Task Arguments (Usage Spec)

```bash
#!/usr/bin/env bash
#USAGE arg "<file>" help="Input file"
#USAGE flag "--verbose" help="Verbose mode"

echo "Processing ${usage_file}"
```

Run with: `mise run process input.txt --verbose`

## Monorepo (Experimental)

Requires `experimental_monorepo_root = true` in root `mise.toml`:

```bash
mise run //packages/api:build    # Absolute path from root
mise run :build                 # Current config_root
mise run '//...:test'           # All projects
```

## Environment Integration

Tasks inherit `[env]` values:

```toml
[env]
DATABASE_URL = "postgresql://localhost/mydb"
_.file = { path = ".env.secrets", redact = true }
```

## Anti-Patterns

| Anti-Pattern | Instead |
|--------------|---------|
| Minimal description | Write: what it does, requires, produces |
| Hardcode secrets | Use `_.file` with `redact = true` |
| Giant monolithic tasks | Break into small tasks with depends |
| Publish without build depends | Add `depends = ["build"]` |

## Remote Task Includes

```toml
[task_config]
includes = [
  "git::https://github.com/org/shared-tasks.git//tasks?ref=main",
  ".mise/tasks",
]
```

See references/arguments.md for usage spec details.
