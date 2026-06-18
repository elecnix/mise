---
name: mise
description: Master mise task runner, tool/version management, and environment configuration. Use when defining, running, debugging, or creating tasks; managing dev tool versions; configuring project environments; working with file tasks; setting up mise for development projects; or troubleshooting task execution, dependencies, or configuration.
license: MIT
compatibility: Requires mise installed (curl https://mise.run | sh). Works on Linux, macOS, and Windows (via WSL or native).
metadata:
  version: "1.0"
  author: elecnix
---

# mise Task Runner & Development Environment

A comprehensive guide to using mise for development environment management, task running, and tool versioning. This skill helps you master mise's task system, troubleshoot common issues, and implement best practices.

## Quick Reference

| Feature | Command |
|---------|---------|
| Install tool | `mise use node@20` (project) or `mise use -g node@20` (global) |
| Run task | `mise run task-name` |
| List tasks | `mise tasks` |
| Edit task | `mise tasks edit task-name` |

## Three Pillars of mise

### 1. Dev Tools Management
```bash
mise use node@20    # Install node 20 in project
mise use -g node@lts # Set global default to LTS
mise install         # Install all tools from config
mise ls              # List installed versions
mise ls-remote python # See available versions
```

### 2. Environment Management
```bash
mise set NODE_ENV=development  # Set env var
mise env                       # Show current env
```

Config in `mise.toml`:
```toml
[env]
_.file = ".env"
DATABASE_URL = "postgresql://localhost/myapp"
```

### 3. Task Runner
```bash
mise run build       # Run 'build' task
mise run test -- -v  # Pass -v to test task
mise watch           # Auto-run on file changes
```

## Task Configuration

### TOML Tasks
```toml
[tasks.build]
run = "npm run build"
description = "Build the project"
alias = "b"
depends = ["lint"]
dir = "{{cwd}}"
quiet = true
```

### File Tasks (Recommended)
Create executable scripts in `.mise/tasks/`:

```bash
#!/usr/bin/env bash
#MISE description="Build the project"
set -euo pipefail

npm run build
```

```bash
chmod +x .mise/tasks/build
mise run build
```

## Task Arguments with Usage

```bash
#!/usr/bin/env bash
#USAGE arg "<file>" help="File to process"
#USAGE flag "--verbose" help="Verbose mode"

echo "Processing ${usage_file}"
```

## Environment Variables Available in Tasks

- `MISE_ORIGINAL_CWD` - Where you ran `mise run`
- `MISE_CONFIG_ROOT` - Directory containing `mise.toml`
- `MISE_TASK_NAME` - Name of the task being run
- `MISE_TASK_DIR` - Directory containing the task script

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Task not found | `mise tasks` to list; ensure file is executable |
| Tools not activating | `mise trust .mise.toml`; `mise install` |
| Parallel conflicts | Add `raw = true` to task |
| File watch not working | Check `sources` array; run `mise watch -v` |

See references/REFERENCE.md for complete configuration reference.
