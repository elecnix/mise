---
name: mise
description: mise task runner - defines and executes project automation tasks with dependency resolution, argument parsing, file-change caching, and environment inheritance via mise.toml [tasks] sections or executable scripts in .mise/tasks/.
license: MIT
compatibility: Requires mise CLI (curl https://mise.run | sh). Linux, macOS, Windows.
metadata:
  version: "1.0"
author: elecnix
---

# mise

mise is a dev-environment manager: tool version pinning (`[tools]`), environment variables (`[env]`), and a task runner (`[tasks]` / `.mise/tasks/`). This skill covers the task runner; tool/env management is outside scope.

## Task Sources

Tasks are defined in two ways, both discovered by walking from cwd up to the config root and merging in order (parent configs first, nearest wins):

- **TOML tasks**: `[tasks.<name>]` tables inside any `mise.toml` / `.mise.toml` / `.mise.local.toml`.
- **File tasks**: executable scripts under `.mise/tasks/`, `mise-tasks/`, or a configured includes dir. The filename (sans extension) is the task name; nested dirs become colon-separated namespaces (`db/migrate/up` → `db:migrate:up`). File tasks support full editor tooling (syntax highlighting, linting) and are preferred over TOML for non-trivial logic.

Tasks from all sources merge into one namespace. Name collisions: file tasks override TOML tasks of the same name.

## Running Tasks

`mise run <task>` executes a task after resolving its dependency DAG (dependencies run first, in parallel when independent). `mise run a b` runs sequentially; `mise run 'a ::: b'` runs in parallel. `mise run '<glob>'` expands namespaces (`test:*`, `db:**`). `mise r` is the short alias. `mise tasks` lists all non-hidden tasks; `mise tasks --hidden` includes tasks prefixed with `_` (convention for internal helpers). `mise tasks edit <name>` opens the task file in `$EDITOR`, creating it if missing. `mise watch <task>` re-runs on `sources` file changes.

## Dependency Resolution

Three dependency edges, all accept arrays of task names:

- `depends`: runs before the task; added to the execution list so they appear in the DAG and run once even if shared.
- `depends_post`: runs after the task succeeds; also added to the execution list.
- `wait_for`: if the named task is already in the current run's execution list, wait for it; otherwise skip. Does NOT add the task to the list (unlike `depends`). Used for soft ordering against optional concurrent work.

Circular dependencies error at resolution time. Dependencies inherit the parent task's environment unless the dependency declares its own `env`.

## Task Properties

| Property | Type | Behavior |
|----------|------|----------|
| `run` | string \| [string] | Command(s) to execute. Array runs sequentially; a multi-line string is one shell invocation. Required. |
| `run_windows` | string | Overrides `run` on Windows. |
| `description` | string | Shown in `mise tasks`. The only context an agent sees when listing tasks — state what it does, what it requires, what it produces, and when to run it. |
| `alias` | string | Additional name to invoke the task by. |
| `dir` | string | Working directory for `run`. Defaults to config root (the directory holding the `mise.toml`). Template-expanded. |
| `depends` / `depends_post` / `wait_for` | [string] | See Dependency Resolution. |
| `env` | {K=V} | Environment variables scoped to this task only. NOT propagated to dependencies. |
| `tools` | {tool=version} | Tool versions activated for this task's execution only. |
| `sources` | [glob] | Input file globs for cache invalidation and `mise watch`. Relative to config root. |
| `outputs` | [glob] | Output artifacts. If all outputs are newer than all sources, the task is skipped (cached). |
| `hide` | bool | Excludes from `mise tasks` default listing. |
| `quiet` | bool | Suppresses mise's `[task] $ cmd` echo line. |
| `silent` | bool | Suppresses all task output (stdout+stderr). |
| `raw` | bool | Connects task stdin/stdout/stderr directly to the parent, disabling parallel execution for this task. Required for prompts (`read`) and interactive tools. |
| `raw_args` | bool | Passes all CLI args through without usage-spec parsing. |
| `interactive` | bool | Takes an exclusive I/O lock; blocks other interactive tasks. |
| `confirm` | string \| {msg} | Prompts for confirmation before running. |
| `shell` | string | Shell used to run `run` (default `sh -c`). e.g. `pwsh -c`. |
| `usage` | spec | Inline usage-spec declaration for TOML tasks (file tasks use `#USAGE` comments; see Arguments). |

## File Task Headers

File tasks declare metadata via leading `#MISE key=value` comments (parsed, not shell):

`#MISE description="…"` · `#MISE alias="…"` · `#MISE depends=["…"]` · `#MISE depends_post=["…"]` · `#MISE wait_for=["…"]` · `#MISE sources=["glob"]` · `#MISE outputs=["glob"]` · `#MISE dir="…"` · `#MISE env={K=V}` · `#MISE hide=true` · `#MISE quiet=true` · `#MISE silent=true` · `#MISE raw=true` · `#MISE shell="…"` · `#MISE tools={…}` · `#MISE confirm="…"`

Values are TOML literals (quoted strings, arrays, objects).

## Arguments (Usage Spec)

Both file tasks (`#USAGE` comments) and TOML tasks (`usage` key, or `raw_args=false`) parse CLI args via the usage spec. The parser exposes values as `usage_<name>` env vars (hyphens → underscores) inside the task.

- `#USAGE arg "<name>" [type=…] [default=…] [choices=[…]] [help="…"]` — positional. `required` unless `default` set.
- `#USAGE flag "--long [-short] [value]" [default=…] [help="…"]` — boolean flag, or value-taking flag when a placeholder is given.
- `#USAGE arg "<name>" required` — mark required.
- `type=` supports `string` (default), `int`, `float`, `bool`, `path`, `file`, `dir`.
- `choices=[…]` constrains to an enum.
- Access: `${usage_<name>}`; flags default to `false` when absent (use `${usage_verbose:-false}`).
- `mise run task -- <args>` passes everything after `--` verbatim when `raw_args` is set.

## Environment Inheritance

Every task inherits the active `[env]` block (including `_.file`-loaded `.env` contents and `_.command` output) plus mise's computed environment. `_.file = { path = ".env", redact = true }` redacts values from `mise env` output to avoid leaking secrets. Task-local `env` merges on top of inherited env. `env` is NOT forwarded to dependency tasks — each dependency re-derives from `[env]`.

## Runtime Variables

mise sets these for each task process:

| Var | Meaning |
|-----|---------|
| `MISE_ORIGINAL_CWD` | Directory where `mise run` was invoked (may differ from config root). |
| `MISE_CONFIG_ROOT` | Directory containing the resolved `mise.toml`. |
| `MISE_TASK_NAME` | Name of the executing task. |
| `MISE_TASK_DIR` | Directory containing the file-task script (file tasks only). |

## Templates

`run`, `dir`, `env` values, and `description` are Tera-template-expanded with: `{{cwd}}`, `{{config_root}}`, `{{env.VAR}}`, `{{vars.name}}` (from `[vars]`), `{{usage.<arg>}}`. Use `${var:-default}` for shell-level defaults inside `run`.

## Monorepo (Experimental)

Enable with `MISE_EXPERIMENTAL=1` env plus `experimental_monorepo_root = true` in the root config. Tasks in nested `mise.toml` files are auto-discovered and namespaced by their path relative to root (`packages/api/.mise.toml` `[tasks.build]` → `packages/api:build`). Invocation forms: `//abs/path:task` (absolute from root), `:task` (current config_root), `//...:task` (all projects), `path/*:task` (glob over subdirs).

## Remote Includes

`[task_config] includes = […]` pulls task definitions from remote or local sources. Each entry is a directory; remote form: `git::https://github.com/org/repo.git//tasks?ref=main`. Later entries override earlier ones; a local `.mise/tasks` entry last lets you override shared remote tasks per-project.

## Pitfalls

- `env` on a task does NOT flow into its `depends` — redeclare or hoist into `[env]`.
- File tasks must be `chmod +x` or they're silently skipped.
- `sources`/`outputs` globs are relative to config root, not cwd.
- `raw=true` is required for any task that reads stdin or needs a TTY; without it the task gets an empty/closed stdin and may hang or fail.
- `depends` tasks run once per `mise run` invocation even if shared across multiple requested tasks; `wait_for` does not trigger a run, only joins one already scheduled.
- Hidden tasks (`_` prefix or `hide=true`) are still runnable explicitly; `hide` only affects `mise tasks` listing.
