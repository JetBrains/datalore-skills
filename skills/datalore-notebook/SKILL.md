---
name: datalore-notebook
description: >
  Work with Datalore workspaces and notebooks through the bundled CLI. Use workspace
  commands for workspace-scoped resources and notebook commands for notebook-scoped
  content such as cells, reports, files, resource attachments, kernels, and definitions.
  Requires uv.
compatibility: >
  Requires macOS/Linux with uv installed. Authenticates with
  DATALORE_API_TOKEN, OAuth PKCE browser login, or per-scope system keychain credentials via the bundled scripts/datalore CLI.
metadata:
  author: JetBrains
  version: 2026.3
---

# Datalore Notebook Agent

Use the bundled `scripts/datalore` CLI for Datalore workspace and notebook work. It is a `uv run --script` CLI with pinned inline Python dependencies,
wraps the public notebook/user/workspace APIs, stores session context in `.datalore-session`, and authenticates with `DATALORE_API_TOKEN` first,
otherwise system keychain credentials saved per scope. When no valid credential is available, `init` starts OAuth PKCE browser login and saves the
resulting credentials.

Run every CLI command with host execution (outside the sandbox) on the first attempt, including `--help` and commands using `DATALORE_API_TOKEN`. The
CLI performs a read-only system-keychain probe at startup and cannot run without keychain access. Do not set `UV_CACHE_DIR` or retry inside the
sandbox; request host execution for the same command instead.

## Mandatory tool routing

For any Datalore URL, workspace, notebook, file, database, cell, or token task, use the bundled `tools/skills/datalore-notebook/scripts/datalore` CLI.
Do not use `curl`, `wget`, raw `requests`, or hand-written HTTP calls unless the user explicitly asks for raw API debugging or the CLI lacks the
required operation.

If a command fails for auth or scope, run the appropriate `datalore init ... --scope ...` command from the current task workspace. Do not bypass the
CLI with `curl` to work around auth.

## Available scripts

- `scripts/datalore` - CLI for Datalore notebook, user, and workspace APIs. The top-level command sets are `init`, `workspaces`, `workspace`,
  `notebook`, and `logout`. Workspace resource commands live under `workspace`; notebook content and notebook attachment commands live under
  `notebook`. For setup, run the absolute path to `scripts/datalore init <url> --scope <scope>` from the task workspace so `.datalore-session` is
  created there. The URL depends on scope:
  notebook URL for `notebook_api` and `notebook_with_workspace_view`, workspace URL for `workspace`, and host root URL for `user_basic`. Workspace
  sessions also take `--access read|create|full` and may retain notebook context with `--notebook-id <ownerId>/<notebookId>`. To remove local access
  for the current session, run `scripts/datalore logout`; to remove credentials for another target, run
  `scripts/datalore logout <url> --scope <scope>`.

## Setup

Initialize from the workspace where the agent will work on this notebook. The CLI writes `.datalore-session` to the current directory, so do not run
`init` from the installed skill directory unless that is intentionally the session workspace.

```bash
# The token avoids credential reads/writes after the mandatory startup keychain probe.
export DATALORE_API_TOKEN="<token>"

cd <task-workspace>
# From the repository root:
tools/skills/datalore-notebook/scripts/datalore init <notebook-url> --scope notebook_api
tools/skills/datalore-notebook/scripts/datalore init <notebook-url> --scope notebook_with_workspace_view  # minimum scope for attachment writes
tools/skills/datalore-notebook/scripts/datalore init <host-root-url> --scope user_basic
tools/skills/datalore-notebook/scripts/datalore init <workspace-url> --scope workspace --access read --workspace-id <owner/workspace>    # for workspace reads: info, entries, list/search/get/usage on resources
tools/skills/datalore-notebook/scripts/datalore init <workspace-url> --scope workspace --access create --workspace-id <owner/workspace> --notebook-id <owner/notebook>  # optionally retain notebook context
tools/skills/datalore-notebook/scripts/datalore init <workspace-url> --scope workspace --access full --workspace-id <owner/workspace>    # for database queries, resource create/update/delete/clone, and destructive workspace ops
```

If `DATALORE_API_TOKEN` is not set, `init` starts OAuth PKCE browser login and saves the resulting token in the keychain under the notebook path. Run
`init` yourself from the task workspace and relay the printed browser URL if the browser does not open automatically. Do not ask the user to paste
tokens into chat.

Before initializing scopes added after Datalore 2026.2, the CLI reads `/api/agent/v1/version`, then calls `/api/agent/v1/validate_version` to validate
compatibility. This preflight runs for supplied API tokens as well as OAuth. If the version endpoint is absent, it probes the existing public notebook
API for legacy agent-skill version support. Workspace initialization also requires the API response's `X-Agent-Skill-Version` to exactly match the
running skill before saving the session. If compatibility cannot be established, initialization stops. For Datalore 2026.2.x, install
datalore-notebook skill version 2026.2.2 instead, or upgrade Datalore to 2026.3 or newer.

If the CLI reports that the installed skill is incompatible with the Datalore deployment, stop and tell the user to align the skill and Datalore
versions. Keep the response to that diagnosis and the two remedies printed by the CLI. Do not reinterpret it as a login, missing-header, or URL-shape
problem, retry with a legacy scope, or include transport-level details unless the user asks for debugging information.

If a valid command fails for authentication, run `init` yourself FROM THE CURRENT DIRECTORY. Do not continue with token discovery. Ask the user to run
`init` manually only if the CLI cannot start OAuth, the OAuth callback times out, keychain access fails, or the user needs to complete browser
authorization outside the agent environment.

Put `--json` before the command for machine-readable responses. `file read` and `file download` always stream raw file bytes to stdout, so do not use
`--json` with them. Prefer the CLI over raw API calls; use `references/notebook-api-reference.md` for notebook endpoints and
`references/workspace-api-reference.md` for user/workspace endpoints when you need schema details.

Authorization requirements are minimums, not exact matches. A broader scope covers every narrower scope below it:
`user_full` > `workspace` > `notebook_with_workspace_view` > `notebook_api` > `user_basic`. Likewise, higher access covers lower access:
`full` > `create` > `read`. Use the least privilege practical for the task, but reuse a broader existing session when it covers the command. Commands
that target a notebook still need notebook identity; when a broader session has no notebook context, pass `--notebook-id <ownerId>/<notebookId>`. The
option applies to the whole notebook command group: `datalore notebook --notebook-id <ownerId>/<notebookId> <command> ...`. Resource attachment
commands also accept it after the attachment subcommand for convenience. Likewise, use
`datalore workspace --workspace-id <ownerId>/<workspaceId> <command> ...` to override the workspace context for a workspace command. Workspace API
requests always send `workspaceId` explicitly; the server does not infer it from the token's resource filter.

The OAuth consent page may offer a danger-styled **Allow full access** action. Choosing it replaces the requested token shape with unrestricted
`USER_FULL` scope and `FULL` access. This token satisfies every CLI user, notebook, and workspace API authorization check, but not admin-only APIs.
The CLI deliberately keeps `.datalore-session` at the scope and target requested by `init`, even when the issued token is broader, so target identity
and command routing remain available. Use `--notebook-id` or `--workspace-id` on the corresponding command group to target another resource with the
broader token; do not edit `.datalore-session` by hand.

## List workspaces

When the user asks to list workspaces on a Datalore URL, derive the host root URL (`https://<host>/`) from the supplied URL and use the CLI:

```bash
tools/skills/datalore-notebook/scripts/datalore init <host-root-url> --scope user_basic
tools/skills/datalore-notebook/scripts/datalore --json workspaces
```

Do not call `/api/user_public/v1/workspaces` with `curl` directly.

## Workflow

1. Orient first, treating authentication as a hard gate. If no `.datalore-session` exists or authentication fails, run `init` FROM THE CURRENT
   DIRECTORY. `init` can start OAuth PKCE browser login and prints an authorization URL before waiting for the callback, so do not stop just because
   the environment is non-interactive. Stop and ask the user to run `init` manually only after the automatic `init` attempt fails in a way the agent
   cannot complete.

```bash
tools/skills/datalore-notebook/scripts/datalore notebook cells --full
tools/skills/datalore-notebook/scripts/datalore --json notebook cells --include-outputs
```

2. Discover data before coding. Do not guess file formats, table names, or columns.

```bash
tools/skills/datalore-notebook/scripts/datalore notebook files --directory data/notebook_files
tools/skills/datalore-notebook/scripts/datalore notebook databases attached
tools/skills/datalore-notebook/scripts/datalore notebook db condensed <databaseId>
tools/skills/datalore-notebook/scripts/datalore notebook db schema <databaseId> --depth 1
tools/skills/datalore-notebook/scripts/datalore notebook db search <databaseId> <query>
```

3. Create/edit cells using stdin for multi-line source.

```bash
tools/skills/datalore-notebook/scripts/datalore notebook cell create --type CODE --after <cell-id> --wait << 'EOF'
print("hello")
EOF

tools/skills/datalore-notebook/scripts/datalore notebook cell edit <cell-id> --wait << 'EOF'
print("updated")
EOF
```

For SQL cells:

```bash
tools/skills/datalore-notebook/scripts/datalore notebook cell create --type SQL --database-id <databaseId> --variable result_df --wait << 'EOF'
select *
from public.orders
limit 10
EOF
```

For markdown:

```bash
tools/skills/datalore-notebook/scripts/datalore notebook cell create --type MARKDOWN --before <first-cell-id> << 'EOF'
# Analysis
EOF
```

4. Run, inspect, and recover.

```bash
tools/skills/datalore-notebook/scripts/datalore notebook cell run <cell-id> --wait
tools/skills/datalore-notebook/scripts/datalore notebook cell outputs <cell-id>
tools/skills/datalore-notebook/scripts/datalore notebook kernel status
tools/skills/datalore-notebook/scripts/datalore notebook kernel interrupt
tools/skills/datalore-notebook/scripts/datalore notebook agent stop
```

Treat `VALID` and `WARNING` as success. For `ERROR`, inspect traceback/stdout/stderr, fix, and rerun. For `TIMEOUT`, the cell may still be running;
use `--wait` for single-cell commands when you need longer polling, and for `cells run` inspect outputs or interrupt before continuing. In non-obvious
cases, when using `--json` for commands that execute cells, check the returned execution status rather than relying only on process exit code. After
errors, assume kernel state may be partially mutated.

Use `notebook agent stop` when the notebook agent itself should be released. Unlike `kernel interrupt`, it stops the whole computation and clears the
kernel state.

5. Clean up before finishing. Only delete cells or files created during the current task.

```bash
tools/skills/datalore-notebook/scripts/datalore notebook cells --full
tools/skills/datalore-notebook/scripts/datalore notebook cell delete <cell-id>
tools/skills/datalore-notebook/scripts/datalore notebook file delete <path>
```

## Commands

```bash
tools/skills/datalore-notebook/scripts/datalore --json <command> ...
tools/skills/datalore-notebook/scripts/datalore logout [notebook-url]
tools/skills/datalore-notebook/scripts/datalore workspaces
tools/skills/datalore-notebook/scripts/datalore workspace info
tools/skills/datalore-notebook/scripts/datalore workspace --workspace-id <ownerId>/<workspaceId> info
tools/skills/datalore-notebook/scripts/datalore workspace entries
tools/skills/datalore-notebook/scripts/datalore workspace reports
tools/skills/datalore-notebook/scripts/datalore workspace file <entity-id>
tools/skills/datalore-notebook/scripts/datalore workspace breadcrumbs <entity-id>
tools/skills/datalore-notebook/scripts/datalore workspace folder <folder-id>
tools/skills/datalore-notebook/scripts/datalore workspace folder-entries <folder-id>
tools/skills/datalore-notebook/scripts/datalore workspace mkdir <name> [--parent <folder-id>]          # requires --access create or full
tools/skills/datalore-notebook/scripts/datalore workspace create-notebook <name> [--parent <folder-id>] [--language python]  # requires --access create or full
tools/skills/datalore-notebook/scripts/datalore workspace rename <entity-id> <name>
tools/skills/datalore-notebook/scripts/datalore workspace move <entity-id> <parent-id>
tools/skills/datalore-notebook/scripts/datalore workspace move-to-trash <entity-id>
tools/skills/datalore-notebook/scripts/datalore workspace remove-from-trash <entity-id>
tools/skills/datalore-notebook/scripts/datalore workspace restore-from-trash <entity-id>
tools/skills/datalore-notebook/scripts/datalore workspace empty-trash
tools/skills/datalore-notebook/scripts/datalore workspace database query --workspace-id <ownerId>/<workspaceId> --database-id <database-id> --name <query-name> --query "SELECT 1"  # streams CSV; requires --access full
tools/skills/datalore-notebook/scripts/datalore workspace databases drivers
tools/skills/datalore-notebook/scripts/datalore notebook cell get <cell-id>
tools/skills/datalore-notebook/scripts/datalore notebook --notebook-id <ownerId>/<notebookId> cells --full
tools/skills/datalore-notebook/scripts/datalore notebook cells run <cell-id>...
tools/skills/datalore-notebook/scripts/datalore notebook files --directory data/notebook_files
tools/skills/datalore-notebook/scripts/datalore notebook file text <path> --max-lines 20
tools/skills/datalore-notebook/scripts/datalore notebook file read <path>  # raw bytes; no --json
tools/skills/datalore-notebook/scripts/datalore notebook file upload <local-path> --directory data/notebook_files
tools/skills/datalore-notebook/scripts/datalore notebook cell create --type CONTROL --control-json '{"controlType":"TEXT_INPUT","label":"Name","variable":"name","value":"Alice","multiline":false}' --wait
tools/skills/datalore-notebook/scripts/datalore notebook cell update-control <cell-id> --control-json '{"controlType":"TEXT_INPUT","label":"Name","variable":"name","value":"Bob","multiline":false}' --wait
tools/skills/datalore-notebook/scripts/datalore notebook worksheet create --name "Analysis"
tools/skills/datalore-notebook/scripts/datalore notebook report get
tools/skills/datalore-notebook/scripts/datalore notebook report add-tab --name "Overview" --tab-id overview
tools/skills/datalore-notebook/scripts/datalore notebook report add-row overview --row-id overview-row
tools/skills/datalore-notebook/scripts/datalore notebook report upsert-cell overview-row --cell-json '{"cellId":"<cell-id>","column":0,"columnSpan":24}'
tools/skills/datalore-notebook/scripts/datalore notebook report publish --type STATIC --mode CREATE
tools/skills/datalore-notebook/scripts/datalore notebook definitions <cell-id> <line> <column>
```

`notebook ...` commands require at least their documented notebook scope, while `workspace ...` commands require at least workspace scope. Broader
scopes and higher access levels satisfy narrower/lower requirements. `workspace database query` requires Full workspace access and streams raw CSV, so
do not use `--json` with it. `worksheet create` is persistent; use it only when a new tab is intended.

## Notebook Reports

Use `notebook report get` to inspect the current layout before changing it. It returns stable tab and row IDs needed by the granular commands. Use a
distinct command for every layout operation—`add-tab`, `rename-tab`, `move-tab`, `remove-tab`, `add-row`, `move-row`, `remove-row`,
`upsert-cell`, and `remove-cell`—rather than constructing raw operation batches. Every mutation is applied atomically through the public report
operations API and returns the materialized layout. `replace --tabs-json <array>` replaces the entire layout; use it only when the intended layout is
complete, as omitted tabs, rows, and cells are removed.

Report reads require `notebook_api` scope and report changes/publishing require Full access when using a workspace-scoped session. A report cell must
refer to an existing notebook cell. Layout uses a 48-column horizontal grid and 20-pixel vertical grid steps. Prefer dynamic height by omitting
`height`; set a fixed `height` of at least 3 grid steps only when fixed sizing or vertical stacking is requested. Fixed-height cells must not overlap.
Pass JSON values inline, as `@file`, or as `-` to read from stdin.

```bash
# Inspect the layout, then add a tab and row with caller-controlled IDs.
tools/skills/datalore-notebook/scripts/datalore --json notebook report get
tools/skills/datalore-notebook/scripts/datalore notebook report add-tab --name "Overview" --tab-id overview
tools/skills/datalore-notebook/scripts/datalore notebook report add-row overview --row-id overview-row

# Place a dynamically sized notebook cell in the first half of the row.
tools/skills/datalore-notebook/scripts/datalore notebook report upsert-cell overview-row --cell-json '{"cellId":"<cell-id>","column":0,"columnSpan":24}'

# Move or remove an existing part of the layout.
tools/skills/datalore-notebook/scripts/datalore notebook report move-row <row-id> <destination-tab-id> 0
tools/skills/datalore-notebook/scripts/datalore notebook report remove-cell <cell-id>

# Replace the complete layout from a JSON array and publish it after inspecting the response.
tools/skills/datalore-notebook/scripts/datalore notebook report replace --tabs-json @report-layout.json
tools/skills/datalore-notebook/scripts/datalore notebook report publish --type INTERACTIVE --mode CREATE --computation-mode REACTIVE
```

Publishing requires an initialized, non-empty layout. Use `--mode UPDATE` only to update an existing publication of the selected type. Static and
interactive publication both accept `--full-width` / `--no-full-width` and `--can-download-and-edit-copy`; `--computation-mode` applies to interactive
reports.

## Notebook Resource Attachments

Manage notebook attachments for attachable resource types through `notebook <resource_type> ...`. Use
`notebook <resource_type> attached` and `notebook <resource_type> get-attached` to inspect current notebook attachments, and use
`notebook <resource_type> attach`, `notebook <resource_type> detach`, and `notebook <resource_type> update-attachment` for resource API writes.
Attachment changes require at least `notebook_with_workspace_view` scope and Create access; attachment reads require at least `notebook_api` scope and
Read access. A `workspace` session therefore covers both when its access is sufficient. These commands use the notebook stored in a notebook-scoped
session by default. For a broader session without notebook context, pass `--notebook-id <ownerId>/<notebookId>`. Prefer these commands over the legacy
`notebook attached-databases` alias when attachment management is the task.

Attachment support by resource type:

- `databases`: `attached`, `get-attached`, `attach`, `detach`
- `datasources`: `attached`, `get-attached`, `attach`, `update-attachment`, `detach`
- `environments`: `attached`, `get-attached`, `attach`, `detach`
- `git-repositories`: `attached`, `get-attached`, `attach`, `update-attachment`, `detach`
- `ssh-keys`: no notebook attachment support

```bash
# List databases attached to the current notebook through the resource API
tools/skills/datalore-notebook/scripts/datalore notebook databases attached

# Attach a database to the current notebook
tools/skills/datalore-notebook/scripts/datalore notebook databases attach <database-id>

# Attach using a workspace-scoped session with Create or Full access
tools/skills/datalore-notebook/scripts/datalore notebook databases attach <database-id> --notebook-id <ownerId>/<notebookId>

# Attach a datasource with explicit mount settings
tools/skills/datalore-notebook/scripts/datalore notebook datasources attach <datasource-id> --attachment-options '{"mountpoint":"raw_data","passToEnv":false,"readOnly":false}'

# Update attached datasource mount settings
tools/skills/datalore-notebook/scripts/datalore notebook datasources update-attachment <datasource-id> --attachment-options '{"mountpoint":"raw_data","passToEnv":true,"readOnly":true}'

# Attach a git repository to the current notebook at a specific reference
tools/skills/datalore-notebook/scripts/datalore notebook git-repositories attach <repo-id> --attachment-options '{"reference":{"reference":"main","type":"Branch"}}'
```

Keep using `notebook db ...` only for SQL schema exploration of already attached databases.

## Workspace Resources

Manage workspace-level resources (databases, datasources, environments, git-repositories, ssh-keys) with type-specific commands. Database resources
support `drivers`, `list`, `get`, `create`, `update`, `delete`, `search`, `usage`, and `clone`; use `workspace database query` to execute SQL against
a workspace database and stream the result as CSV. The other resource types support `list`, `get`, `create`, `update`, `delete`, `search`, `usage`,
and `clone`. Use a workspace session with `--access read` for `list/get/search/usage`. Use `--access full` for database queries and
`create/update/delete/clone`.

Before creating a workspace database resource, always fetch the available database drivers from Datalore first and map the user's requested database
type to a returned driver. Do not guess `--driver-id` from memory or by string similarity alone. Resolve names like `postgres`, `postgresql`,
`snowflake`, `bigquery`, `mysql`, `sql server`, and `mssql` against the live driver list, then pass the chosen returned driver ID to
`datalore workspace databases create --driver-id ...`.

```bash
# List available database drivers before database creation
tools/skills/datalore-notebook/scripts/datalore --json workspace databases drivers

# List workspace databases
tools/skills/datalore-notebook/scripts/datalore workspace databases list

# Get a database by ID
tools/skills/datalore-notebook/scripts/datalore workspace databases get <database-id>

# Execute a query and stream the raw CSV result to stdout (requires Full workspace access)
tools/skills/datalore-notebook/scripts/datalore workspace database query --workspace-id <ownerId>/<workspaceId> --database-id <database-id> --name "Recent orders" --query "SELECT * FROM orders LIMIT 10"

# Search databases by name
tools/skills/datalore-notebook/scripts/datalore workspace databases search "postgres"

# List datasources
tools/skills/datalore-notebook/scripts/datalore workspace datasources list

# List environments
tools/skills/datalore-notebook/scripts/datalore workspace environments list

# List git repositories
tools/skills/datalore-notebook/scripts/datalore workspace git-repositories list

# List SSH keys
tools/skills/datalore-notebook/scripts/datalore workspace ssh-keys list

# Clone a database to current workspace
tools/skills/datalore-notebook/scripts/datalore workspace databases clone <database-id> --new-name "My Copy"

# See where a resource is used
tools/skills/datalore-notebook/scripts/datalore workspace databases usage <database-id>
```

CONTROL cells are executable in single-cell commands (`notebook cell create --type CONTROL --wait`, `notebook cell run`, and
`notebook cell update-control --wait`) because running them assigns the configured value to the kernel variable. Batch `notebook cells run` is limited
to CODE and SQL cells.

## Troubleshooting

- Incompatible skill and Datalore versions: stop and tell the user to install datalore-notebook skill version 2026.2.2 for Datalore 2026.2.x, or
  upgrade Datalore to 2026.3 or newer. Do not present the compatibility probe's HTTP status, response headers, or URL requirements unless the user
  asks for debugging details.
- Failed compatibility probe without an incompatibility diagnosis: on newer deployments, ensure the reverse proxy passes `/api/agent/v1` without an
  OAuth cookie and preserves `X-Agent-Skill-Version` on public API responses.
- No credentials or 401: Run `tools/skills/datalore-notebook/scripts/datalore init <url> --scope <scope>` from the task workspace, using a notebook
  URL for `notebook_api`, a workspace URL for `workspace`, or the host root URL for `user_basic`. For workspace sessions, add
  `--access read` for workspace reads, `--access create` for `workspace mkdir` / `workspace create-notebook`, and `--access full` for
  `workspace database query`, workspace resource writes, and destructive workspace actions. Add `--notebook-id <ownerId>/<notebookId>` when notebook
  commands will reuse the workspace session. If OAuth PKCE prints an authorization URL, relay it to the user. Ask the user to run the command manually
  only if the agent-run `init` cannot complete.
- Empty/unhelpful output: retry with `--json` or `--verbose`.
- File endpoints fail before computation starts: run
  `tools/skills/datalore-notebook/scripts/datalore notebook cell create --type CODE --wait --source "print('ready')"` and retry.
- Insert at top: use `--before <first-cell-id>`. Omitting `--before`/`--after` appends.
