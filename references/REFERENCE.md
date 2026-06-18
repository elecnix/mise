# mise Reference

## Configuration Options

### `[tools]`
```toml
node = "20"
python = "latest"
go = "lts"
rust = "nightly"
```

### `[env]`
```toml
_.file = ".env"
DATABASE_URL = "postgresql://localhost/myapp"
```

### Task Properties
| Property | Description |
|----------|-------------|
| `run` | Required: command(s) |
| `description` | Shown in listings |
| `alias` | Short name |
| `depends` | Pre-requisite tasks |
| `dir` | Working directory |
| `sources` | Input files for watches |
| `outputs` | Output files for caching |
| `quiet` | Hide command echo |
| `raw` | Direct I/O (blocks parallel) |

## CLI Commands

| Command | Description |
|---------|-------------|
| `mise install` | Install tools |
| `mise use` | Activate and record |
| `mise run` | Execute task |
| `mise tasks` | List tasks |
| `mise watch` | Auto-run on changes |
| `mise set` | Set env var |
| `mise doctor` | Diagnose issues |
