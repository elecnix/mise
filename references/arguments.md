# Task Arguments (Usage Spec)

## Positional Arguments

```bash
#USAGE arg "<file>" help="Input file"
#USAGE arg "<count>" type=int default=5
```

Access: `${usage_file}` (hyphens become underscores)

## Flags

```bash
#USAGE flag "--verbose" help="Verbose mode"
#USAGE flag "-o --output <file>" help="Output file"
```

Access: `${usage_verbose:-false}`

## Choices

```bash
#USAGE arg "<env>" choices=["dev", "staging", "prod"]
```

## Running Tasks with Args

```bash
mise run process input.txt --verbose
mise run deploy --env prod
mise run build -- -c opt
```
