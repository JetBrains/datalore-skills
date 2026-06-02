---
name: datalore-notebook
description: >
  Work with Datalore notebooks through the bundled CLI. Read, write, and execute
  notebook cells to perform data analysis, build visualizations, query databases,
  and manage notebook files. Use when user shares a Datalore notebook URL, asks to
  "analyze data in Datalore", "run a notebook", "explore a dataset", "write SQL
  against attached databases", or "create notebook cells". Requires uv.
compatibility: >
  Requires macOS/Linux with uv installed. Authenticates with
  DATALORE_API_TOKEN or notebook-scoped system keychain credentials via the bundled scripts/datalore CLI.
metadata:
  author: JetBrains
  version: 0.0.1
---

# Datalore Notebook Agent

Use the bundled `scripts/datalore` CLI for notebook work. It is a `uv run --script` CLI with pinned inline Python dependencies, wraps the public Notebook API, stores notebook identity in `.datalore-session`, and authenticates with `DATALORE_API_TOKEN` first, otherwise system keychain credentials saved per notebook path.

## Available scripts

- `scripts/datalore` - CLI for Datalore notebook cells, files, databases, kernel, and worksheet operations. Run notebook commands through this script; for human keychain setup, ask the user to run the absolute path to `scripts/datalore init <notebook-url>` from the task workspace so `.datalore-session` is created there.

## Setup

Initialize from the workspace where the agent will work on this notebook. The CLI writes `.datalore-session` to the current directory, so do not run `init` from the installed skill directory unless that is intentionally the session workspace.

```bash
# keychain-free setups; CLI will not read/write keychain when this is set
export DATALORE_API_TOKEN="<token>"

cd <task-workspace>
# Assuming the skill is installed in ~/.agents. Adjust the absolute path according to your setup.
~/.agents/skills/datalore-notebook/scripts/datalore init <notebook-url>
```

If `DATALORE_API_TOKEN` is not set, `init` may prompt for an API token and save it in the keychain under the notebook path,
so ask the user to run it from the task workspace when needed. Do not ask the user to paste tokens into chat.

If some valid command fails for authentication, stop and ask the user to run init FROM THE CURRENT DIRECTORY. Do not continue with token discovery.
Do not propose the ! prefix (bang-prefix command mode) for this: it's not a TTY.

Put `--json` before the command for machine-readable responses. This applies to API commands that return structured data; `file read` and `file download` always stream raw file bytes to stdout, so do not use `--json` with them. Prefer the CLI over raw API calls; use `references/api-reference.md` only for schema details.

## Workflow

1. Orient first, treating authentication as a hard gate.
If `init` fails for any reason — including a non-interactive/no-TTY environment where it cannot prompt for a token — STOP and ask the user to run `init` FROM THE CURRENT DIRECTORY themselves.
Do not propose the ! prefix (bang-prefix command mode) for this: it's not a TTY.

```bash
scripts/datalore cells --full
scripts/datalore --json cells --include-outputs
```

2. Discover data before coding. Do not guess file formats, table names, or columns.

```bash
scripts/datalore files --directory data/notebook_files
scripts/datalore databases
scripts/datalore db condensed <databaseId>
scripts/datalore db schema <databaseId> --depth 1
scripts/datalore db search <databaseId> <query>
```

3. Create/edit cells using stdin for multi-line source.

```bash
scripts/datalore cell create --type CODE --after <cell-id> --wait << 'EOF'
print("hello")
EOF

scripts/datalore cell edit <cell-id> --wait << 'EOF'
print("updated")
EOF
```

For SQL cells:

```bash
scripts/datalore cell create --type SQL --database-id <databaseId> --variable result_df --wait << 'EOF'
select *
from public.orders
limit 10
EOF
```

For markdown:

```bash
scripts/datalore cell create --type MARKDOWN --before <first-cell-id> << 'EOF'
# Analysis
EOF
```

4. Run, inspect, and recover.

```bash
scripts/datalore cell run <cell-id> --wait
scripts/datalore cell outputs <cell-id>
scripts/datalore kernel status
scripts/datalore kernel interrupt
```

Treat `VALID` and `WARNING` as success. For `ERROR`, inspect traceback/stdout/stderr, fix, and rerun. For `TIMEOUT`, the cell may still be running; use `--wait` for single-cell commands when you need longer polling, and for `cells run` inspect outputs or interrupt before continuing. In non-obvious cases, when using `--json` for commands that execute cells, check the returned execution status rather than relying only on process exit code. After errors, assume kernel state may be partially mutated.

5. Clean up before finishing. Only delete cells or files created during the current task.

```bash
scripts/datalore cells --full
scripts/datalore cell delete <cell-id>
scripts/datalore file delete <path>
```

## Commands

```bash
scripts/datalore --json <command> ...
scripts/datalore cell get <cell-id>
scripts/datalore cells run <cell-id>...
scripts/datalore files --directory data/notebook_files
scripts/datalore file text <path> --max-lines 20
scripts/datalore file read <path>  # raw bytes; no --json
scripts/datalore file upload <local-path> --directory data/notebook_files
scripts/datalore cell create --type CONTROL --control-json '{"controlType":"TEXT_INPUT","label":"Name","variable":"name","value":"Alice","multiline":false}' --wait
scripts/datalore cell update-control <cell-id> --control-json '{"controlType":"TEXT_INPUT","label":"Name","variable":"name","value":"Bob","multiline":false}' --wait
scripts/datalore worksheet create --name "Analysis"
```

`worksheet create` is persistent; use it only when a new tab is intended.

CONTROL cells are executable in single-cell commands (`cell create --type CONTROL --wait`, `cell run`, and `cell update-control --wait`) because running them assigns the configured value to the kernel variable. Batch `cells run` is limited to CODE and SQL cells.

## Troubleshooting

- No credentials or 401: Stop and ask the user to run `/absolute/path/to/datalore-notebook/scripts/datalore init <notebook-url>` from the task workspace.
- Empty/unhelpful output: retry with `--json` or `--verbose`.
- File endpoints fail before computation starts: run `scripts/datalore cell create --type CODE --wait --source "print('ready')"` and retry.
- Insert at top: use `--before <first-cell-id>`. Omitting `--before`/`--after` appends.
