# Datalore Notebook API Reference

All endpoints use `POST` with `Content-Type: application/json` unless noted otherwise.

**Base URL:** `https://<host>/api/notebook/v1` — derive `<host>` from the notebook URL. Default: `datalore.jetbrains.com`.

Every request requires a `notebookId` object: `{"ownerId": "...", "id": "..."}` extracted from the notebook URL.

## Cell types

| Type       | `data` field                                   | Notes                                                                                 |
|------------|------------------------------------------------|---------------------------------------------------------------------------------------|
| `CODE`     | `{"language": "python"}`                       | Languages: `python`, `kotlin`, `scala`, `sql`, `r` (default: `python`)                |
| `MARKDOWN` | `{}`                                           | No additional data                                                                    |
| `SQL`      | `{"databaseId": "...", "variableName": "..."}` | Native SQL against attached DB; result becomes a DataFrame                            |
| `CONTROL`  | `{"control": {...}}`                           | Interactive widgets — exactly one control per cell. See "Control cell payloads" below |

## Control cell payloads

For a `CONTROL` cell, `data.control` carries the widget definition. Every control has `label` (display text) and `variable` (the Python variable the
kernel assigns when the cell runs). Each control has a `controlType` discriminator and type-specific fields:

| `controlType` | Type-specific fields                                                                                                          |
|---------------|-------------------------------------------------------------------------------------------------------------------------------|
| `CHECKBOX`    | `value: boolean`                                                                                                              |
| `SLIDER`      | `value: string`, `min: string`, `max: string`, `step: string` (string-encoded numbers)                                        |
| `TEXT_INPUT`  | `value: string`, `multiline: boolean`                                                                                         |
| `DATEPICKER`  | `startDate: string?` (ISO date), `endDate: string?`, `withRange: boolean`                                                     |
| `DROPDOWN`    | `items: string[]`, `selectedValue: string?`, `multiselect: boolean`, `selectedValues: string[]?`, `selectedVariable: string?` |

**Static vs dynamic dropdown.** A dropdown is **dynamic** when `selectedVariable` is set to the name of a Python variable holding a list/collection —
the kernel resolves the actual items from that variable at run time, and `variable` is assigned the element matching the user's selection. When
`selectedVariable` is `null` (or omitted), the dropdown is **static** and the literal `items` array is used.

Static single-select:

```json
{"controlType": "DROPDOWN", "label": "Color", "variable": "color",
 "items": ["red", "green", "blue"], "selectedValue": "green",
 "multiselect": false, "selectedValues": []}
```

Dynamic single-select (items sourced from `my_items` in the kernel):

```json
{"controlType": "DROPDOWN", "label": "Fruit", "variable": "fruit",
 "items": [], "selectedValue": "banana",
 "multiselect": false, "selectedValues": [],
 "selectedVariable": "my_items"}
```

For a dynamic dropdown, define the source variable in a CODE cell that runs **before** the control cell; `selectedValue` / `selectedValues` should be
elements present in that variable at run time.

For multi-select, set `multiselect: true` and use `selectedValues` instead of `selectedValue`. Multi-select works for both static and dynamic
dropdowns.

## Output types

`TEXT`, `HTML`, `TABLE`, `MARKDOWN`, `SVG`, `LATEX`, `PLOTLY`, `VEGA`, `PLOT`, `LAST_EXPRESSION`, `TSV`

The `text` field is `null` for binary-only types like `PLOT`. Outputs longer than 100,000 characters are truncated (`"truncated": true`).

## Execution statuses

All: `NONE`, `STARTED`, `WAITING`, `PROCESSING_OUTPUT`, `VALID`, `ERROR`, `WARNING`, `TIMEOUT`. Terminal states: `VALID`, `ERROR`, `WARNING`. Treat
`WARNING` as success.

`timeoutMs` is the server-side request wait window, not a total execution deadline. On `TIMEOUT`, the cell may still be running kernel-side;
single-cell `--wait` commands keep polling until the CLI timeout, while `cells run` has no extra polling phase and exits 2 if the batch times out.

CLI `--json` mode prints these API statuses as-is. When automating or debugging execution commands, use `executionStatus`,
`runResult.executionStatus`, or each `results[].executionStatus` to distinguish successful notebook execution from `ERROR` or `TIMEOUT`; a zero
process exit code only means the CLI command and HTTP request succeeded.

## Kernel states

Returned by `/kernel/status` and `/kernel/restart`:

| State         | Meaning                                                                                                                     |
|---------------|-----------------------------------------------------------------------------------------------------------------------------|
| `NOT_STARTED` | No compute session exists for this notebook yet. The first run/restart call will create one.                                |
| `READY`       | Session is up; no cell is currently running.                                                                                |
| `BUSY`        | Session is up; a cell is in `STARTED`, `WAITING`, or `PROCESSING_OUTPUT`. `runningCellId` on the response identifies which. |

Note: transient kernel-lifecycle states (booting after restart, dead after crash) are not currently distinguished — they manifest as `READY` briefly,
or as a failing `/cell/run` with a non-`VALID` status.

## Timing

Every execution-producing response (`/cell/run`, `/cells/run` results, `runResult` from `/cell/create`, `/cell/edit`, `/cell/controls/update`, plus
`/cell` and `/cell/outputs`) carries a `timing` object:

```json
"timing": {
  "durationMs": 12340,
  "lastExecutedAt": "2026-04-21T10:05:00.423Z"
}
```

`timing` is `null` for cells that have never been executed. `durationMs` and `lastExecutedAt` are both `null` individually when the cell started but
never finished (e.g. the cell was still running when the server reloaded outputs).

---

## `POST /cells` — List cells

Returns all cells in a worksheet. Two response shapes depending on `includeOutputs`.

**Request:**

| Field            | Type    | Required | Description                                                                                                                                                    |
|------------------|---------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `notebookId`     | object  | yes      | `{"ownerId": "...", "id": "..."}`                                                                                                                              |
| `worksheetIndex` | integer | no       | Worksheet tab index (default: 0)                                                                                                                               |
| `includeOutputs` | boolean | no       | If `true`, populate `cellsWithOutputs` with full `CellDetail` objects (source + outputs + execution status + timing) and leave `cells` empty. Default: `false` |

**Response (summary, `includeOutputs` omitted or `false`):**

```json
{
  "cells": [
    {
      "cellId": "kN3xQ7",
      "cellType": "CODE",
      "position": 0,
      "source": "import pandas as pd\ndf = pd.read_csv('data.csv')",
      "data": {"language": "python"}
    }
  ],
  "cellsWithOutputs": null,
  "outputsTruncated": false,
  "outputsTruncatedFromCellId": null
}
```

**Response (detailed, `includeOutputs: true`):**

```json
{
    "cells": [],
    "cellsWithOutputs": [
        {
            "cellId": "kN3xQ7",
            "cellType": "CODE",
            "position": 0,
            "source": "print('hello')",
            "executionStatus": "VALID",
            "executionCount": 3,
            "timing": {
                "durationMs": 42,
                "lastExecutedAt": "2026-04-21T10:05:00.123Z"
            },
            "outputs": [
                {
                    "outputType": "TEXT",
                    "text": "hello\n",
                    "truncated": false
                }
            ],
            "data": {
                "language": "python"
            }
        }
    ],
    "outputsTruncated": false,
    "outputsTruncatedFromCellId": null
}
```

**Output size cap.** Detailed responses are capped at ~5 million chars of accumulated output text across all cells. If including a cell's outputs
would push the total past the cap, that cell's outputs are dropped (the outputs that fit still come back). Specifically:

- `outputsTruncated` becomes `true`
- `outputsTruncatedFromCellId` is set to the first cellId whose outputs were dropped
- That cell and all following cells have `outputs: []` in the response — their other fields (source, executionStatus, timing) are still populated

Use `/cell/outputs` on a specific cell to retrieve outputs that were dropped.


---

## `POST /worksheet/create` — Create worksheet

Creates a new worksheet tab. The worksheet starts with one empty CODE cell in the notebook's primary language.

**Request:**

| Field            | Type    | Required | Description                                                        |
|------------------|---------|----------|--------------------------------------------------------------------|
| `notebookId`     | object  | yes      | Notebook identifier                                                |
| `name`           | string  | no       | Worksheet name. If omitted, a unique `Sheet N` name is generated   |
| `worksheetIndex` | integer | no       | Insert at this worksheet tab index. If omitted, appends to the end |

**Response:**

```json
{
  "worksheetId": "wP2sQ9",
  "worksheetIndex": 1,
  "name": "Analysis",
  "firstCellId": "rT5yU8"
}
```

Use `firstCellId` with `/cell/edit`, or as `beforeCellId` / `afterCellId` in `/cell/create`, to populate the new worksheet.

---

## `POST /report` — Get report layout

Returns the notebook's report layout. `initialized` is `false` and `tabs` is empty when no report layout has been created.

**Request:**

```json
{"notebookId": {"ownerId": "alice", "id": "notebook123"}}
```

**Response:**

```json
{
  "initialized": true,
  "tabs": [{
    "tabId": "overview",
    "name": "Overview",
    "rows": [{
      "rowId": "summary-row",
      "cells": [{
        "cellId": "kN3xQ7",
        "column": 0,
        "columnSpan": 24,
        "verticalOffset": 0,
        "showInput": true,
        "showOutput": true
      }]
    }]
  }]
}
```

---

## `POST /report/replace` — Replace report layout

Atomically replaces the complete layout and returns its materialized form. `tabs: []` removes the report. New tabs and rows can omit their IDs; the
server generates them. Every included `cellId` must identify an existing notebook cell.

**Request:**

```json
{
  "notebookId": {"ownerId": "alice", "id": "notebook123"},
  "tabs": [{
    "name": "Overview",
    "rows": [{
      "cells": [{"cellId": "kN3xQ7", "column": 0, "columnSpan": 48}]
    }]
  }]
}
```

The layout fields are:

| Field                     | Applies to | Default / constraints                                                                        |
|---------------------------|------------|----------------------------------------------------------------------------------------------|
| `tabId`, `rowId`          | tabs, rows | Optional on creation; stable IDs are returned by the API                                     |
| `name`                    | tab        | Empty string by default                                                                      |
| `cellId`                  | cell       | Required existing notebook-cell ID; may appear only once in the layout                       |
| `column`                  | cell       | `0`; integer 0–44                                                                            |
| `columnSpan`              | cell       | `48`; integer 4–48 and `column + columnSpan <= 48`                                           |
| `verticalOffset`          | cell       | `0`; non-negative count of 20-pixel grid steps                                               |
| `height`                  | cell       | Prefer omission for dynamic/content-sized height; fixed height must be at least 3 grid steps |
| `inputHeight`             | cell       | Only for fixed height; integer from 2 through `height - 1`                                   |
| `showInput`, `showOutput` | cell       | `true`                                                                                       |

Fixed-height cells cannot overlap within a row. At most one dynamic-height cell is allowed per row, and it cannot be vertically offset.

---

## `POST /report/operations` — Apply report layout operations

Applies ordered operations atomically and returns the materialized layout. Callers can supply IDs in `ADD_TAB` and `ADD_ROW` when a later operation in
the same batch must refer to a newly created item.

**Request:**

```json
{
  "notebookId": {"ownerId": "alice", "id": "notebook123"},
  "operations": [
    {"type": "ADD_TAB", "tabId": "overview", "name": "Overview"},
    {"type": "ADD_ROW", "tabId": "overview", "rowId": "summary-row"},
    {"type": "UPSERT_CELL", "rowId": "summary-row", "cell": {"cellId": "kN3xQ7"}}
  ]
}
```

| `type`        | Required fields                        | Optional fields          |
|---------------|----------------------------------------|--------------------------|
| `ADD_TAB`     |                                        | `tabId`, `name`, `index` |
| `RENAME_TAB`  | `tabId`, `name`                        |                          |
| `MOVE_TAB`    | `tabId`, `index`                       |                          |
| `REMOVE_TAB`  | `tabId`                                |                          |
| `ADD_ROW`     | `tabId`                                | `rowId`, `index`         |
| `MOVE_ROW`    | `rowId`, `tabId`, `index`              |                          |
| `REMOVE_ROW`  | `rowId`                                |                          |
| `UPSERT_CELL` | `rowId`, `cell` (a cell-layout object) |                          |
| `REMOVE_CELL` | `cellId`                               |                          |

Removing a tab or row also removes its cells. Upserting a cell that is already in the layout moves it to the target row. Removing a cell also removes
its now-empty row.

---

## `POST /report/publish` — Create or update a report publication

Publishes an initialized, non-empty report layout. `UPDATE` requires an existing publication of the selected type.

| Field                    | Required | Description                                         |
|--------------------------|----------|-----------------------------------------------------|
| `type`                   | yes      | `STATIC` or `INTERACTIVE`                           |
| `mode`                   | yes      | `CREATE` or `UPDATE`                                |
| `fullWidth`              | no       | Defaults to `true`                                  |
| `computationMode`        | no       | Interactive only: `REACTIVE` (default) or `JUPYTER` |
| `canDownloadAndEditCopy` | no       | Defaults to `false`                                 |

**Response:**

```json
{"reportId": "report123", "type": "INTERACTIVE", "mode": "CREATE"}
```

---

## `POST /cell` — Get cell

Returns a single cell with execution status and outputs.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |
| `cellId`     | string | yes      | Cell ID             |

**Response:**

```json
{
  "cell": {
    "cellId": "kN3xQ7",
    "cellType": "CODE",
    "position": 0,
    "source": "print('hello')",
    "executionStatus": "VALID",
    "executionCount": 3,
    "timing": {"durationMs": 7, "lastExecutedAt": "2026-04-21T10:05:00.123Z"},
    "outputs": [
      {"outputType": "TEXT", "text": "hello\n", "truncated": false}
    ],
    "data": {"language": "python"}
  }
}
```

`timing` is `null` for cells that have never been executed.

---

## `POST /cell/outputs` — Get cell outputs

Returns detailed execution outputs. Use for polling long-running cells.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |
| `cellId`     | string | yes      | Cell ID             |

**Response:**

```json
{
  "cellId": "kN3xQ7",
  "executionStatus": "VALID",
  "executionCount": 5,
  "timing": {"durationMs": 342, "lastExecutedAt": "2026-04-21T10:05:00.543Z"},
  "stdout": "Processing...\n",
  "stderr": null,
  "errorMessage": null,
  "errorTraceback": null,
  "outputs": [
    {"outputType": "TEXT", "text": "Processing...\n", "truncated": false}
  ]
}
```

`stdout`, `stderr`, `errorTraceback` are truncated to 10,000 chars. Boolean flags `stdoutTruncated`, `stderrTruncated`, `errorTracebackTruncated`
indicate when this happens.

---

## `POST /cell/create` — Create cell

Creates a new cell. Optionally runs it immediately.

**Request:**

| Field          | Type    | Required | Description                                                     |
|----------------|---------|----------|-----------------------------------------------------------------|
| `notebookId`   | object  | yes      | Notebook identifier                                             |
| `cellType`     | string  | yes      | `CODE`, `MARKDOWN`, `SQL`, or `CONTROL`                         |
| `source`       | string  | no       | Cell source code (**not** `content`). Ignored for CONTROL cells |
| `data`         | object  | no       | Cell-type-specific data (see Cell types table above)            |
| `afterCellId`  | string  | no       | Insert after this cell. If omitted, appends to end              |
| `beforeCellId` | string  | no       | Insert before this cell. Mutually exclusive with `afterCellId`  |
| `run`          | boolean | no       | Execute after creation                                          |
| `timeoutMs`    | integer | no       | Max wait for execution (default: 30000, max: 30000)             |

**Response:**

```json
{
  "cellId": "rT5yU8",
  "runResult": {
    "cellId": "rT5yU8",
    "executionStatus": "VALID",
    "executionCount": 1,
    "timing": {"durationMs": 88, "lastExecutedAt": "2026-04-21T10:05:01.200Z"},
    "stdout": null,
    "stderr": null,
    "errorMessage": null,
    "errorTraceback": null,
    "outputs": [
      {"outputType": "TABLE", "text": "<table>...</table>", "truncated": false}
    ]
  }
}
```

`runResult` is `null` when `run` is not set or `false`.

---

## `POST /cell/edit` — Edit cell

Replaces cell source code. Optionally runs it.

**Request:**

| Field        | Type    | Required | Description                                                                                                 |
|--------------|---------|----------|-------------------------------------------------------------------------------------------------------------|
| `notebookId` | object  | yes      | Notebook identifier                                                                                         |
| `cellId`     | string  | yes      | Cell ID to edit                                                                                             |
| `newContent` | string  | yes      | New source code (**not** `source`)                                                                          |
| `data`       | object  | no       | SQL cell metadata only: `{"databaseId": "...", "variableName": "..."}`. Rejected with 400 for non-SQL cells |
| `run`        | boolean | no       | Execute after editing                                                                                       |
| `timeoutMs`  | integer | no       | Max wait for execution (default: 30000, max: 30000)                                                         |

**Response:**

```json
{
  "cellId": "kN3xQ7",
  "applied": true,
  "runResult": { ... }
}
```

---

## `POST /cell/delete` — Delete cell

Permanently deletes a cell.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |
| `cellId`     | string | yes      | Cell ID to delete   |

**Response:** `204 No Content`

---

## `POST /cell/run` — Run cell

Executes a cell and waits for completion. Starts the kernel if needed.

**Request:**

| Field        | Type    | Required | Description                                 |
|--------------|---------|----------|---------------------------------------------|
| `notebookId` | object  | yes      | Notebook identifier                         |
| `cellId`     | string  | yes      | Cell ID to execute                          |
| `timeoutMs`  | integer | no       | Max wait in ms (default: 30000, max: 30000) |

**Response:** Same shape as `runResult` in create/edit responses:

```json
{
  "cellId": "kN3xQ7",
  "executionStatus": "VALID",
  "executionCount": 6,
  "timing": {"durationMs": 1245, "lastExecutedAt": "2026-04-21T10:05:04.500Z"},
  "stdout": "Training complete.\n",
  "stderr": null,
  "errorMessage": null,
  "errorTraceback": null,
  "outputs": [
    {"outputType": "TEXT", "text": "Training complete.\n", "truncated": false}
  ]
}
```

On timeout, `executionStatus` will be `TIMEOUT`. The cell is still running kernel-side; poll `/cell/outputs` to check for completion, or call
`/kernel/interrupt` to stop it.

---

## `POST /cells/run` — Run cells (batch)

Runs a sequence of CODE or SQL cells in order, one at a time. Each cell is executed via the same path as `/cell/run`; the per-cell `timeoutMs` applies
to each cell individually.

**Request:**

| Field         | Type     | Required | Description                                                                                              |
|---------------|----------|----------|----------------------------------------------------------------------------------------------------------|
| `notebookId`  | object   | yes      | Notebook identifier                                                                                      |
| `cellIds`     | string[] | yes      | Non-empty list of cell IDs to run in order. All must be CODE or SQL — MARKDOWN/CONTROL rejected with 400 |
| `stopOnError` | boolean  | no       | If `true` (default), stop on the first cell that returns `ERROR`                                         |
| `timeoutMs`   | integer  | no       | Per-cell timeout in ms (default: 30000, max: 30000)                                                      |

**Response:**

```json
{
  "results": [
    {"cellId": "a", "executionStatus": "VALID", "timing": {...}, "outputs": [...], ...},
    {"cellId": "b", "executionStatus": "ERROR", "timing": {...}, "errorMessage": "...", ...}
  ],
  "stopped": true,
  "stoppedReason": "error"
}
```

`results` is as-complete-as-it-got — truncated to the cell that triggered the stop. `stoppedReason`:

| Value       | Cause                                                                                        |
|-------------|----------------------------------------------------------------------------------------------|
| `null`      | All cells ran to completion (may include `ERROR` cells when `stopOnError: false`)            |
| `"error"`   | First `ERROR` encountered with `stopOnError: true`                                           |
| `"timeout"` | A cell exceeded `timeoutMs`. The kernel is still running it — interrupt before the next call |

**400** on: empty `cellIds`, any MARKDOWN or CONTROL cellId in the list.

---

## `POST /cell/controls/update` — Update control

Updates a control cell's widget value. The control type in the payload **must match** the existing control type.

**Request:**

| Field        | Type    | Required | Description                                         |
|--------------|---------|----------|-----------------------------------------------------|
| `notebookId` | object  | yes      | Notebook identifier                                 |
| `cellId`     | string  | yes      | Control cell ID (must have exactly 1 control)       |
| `control`    | object  | yes      | New control data (must match existing control type) |
| `run`        | boolean | no       | Execute after updating                              |
| `timeoutMs`  | integer | no       | Max wait for execution                              |

**Response:** Same as edit — `{"cellId": "...", "applied": true, "runResult": ...}`

---

## `POST /kernel/status` — Kernel state

Reports the current kernel state for a notebook. Lightweight — safe to poll.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |

**Response:**

```json
{"state": "BUSY", "runningCellId": "kN3xQ7"}
```

`runningCellId` is `null` unless `state` is `BUSY`. See the "Kernel states" section for the value set.

---

## `POST /kernel/interrupt` — Interrupt running cell

Sends an interrupt signal to the kernel. Stops whatever cell is currently running. Idempotent and safe to call when idle.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |

**Response:**

```json
{"interrupted": true}
```

`interrupted` is `true` when a cell was actively running at the moment of the call, `false` otherwise. The interrupted cell's next `/cell/outputs`
poll will show `executionStatus: "ERROR"` with a `KeyboardInterrupt` traceback.

**409 Conflict** with `{"error": "...", "state": "NOT_STARTED"}` if no session exists for the notebook — start the kernel via any `/cell/run` first.

---

## `POST /kernel/restart` — Restart kernel

Restarts an already-booted kernel. All variables and imports are wiped; imports must be re-run from scratch. The call can block until the kernel is
ready, or return immediately.

**409 Conflict** with `{"error": "...", "state": "NOT_STARTED"}` if no kernel has been booted for the notebook yet. Call `/cell/run` first to boot the
kernel, then restart as needed.

**Request:**

| Field                  | Type    | Required | Description                                                                        |
|------------------------|---------|----------|------------------------------------------------------------------------------------|
| `notebookId`           | object  | yes      | Notebook identifier                                                                |
| `reinstallEnvironment` | boolean | no       | If `true`, also reinstall the environment (slower). Default: `false`               |
| `waitUntilReady`       | boolean | no       | If `true`, block until the kernel is ready or the wait times out. Default: `false` |
| `waitTimeoutMs`        | integer | no       | Max wait in ms when `waitUntilReady: true`. Default 60000, range 1000–120000       |

**Response:**

```json
{"state": "READY", "timedOut": false}
```

When `waitUntilReady: false`: returns as soon as the restart submission is accepted (a brief probe catches immediate-dispatch failures so 200
genuinely means "restart in flight"). Follow up with `/kernel/status` if you need to confirm the kernel has finished booting.

When `waitUntilReady: true` and the wait timed out: `timedOut: true` with the last observed `state`. The restart is still in progress; do not treat as
a hard failure.

---

## `POST /agent/stop` — Stop notebook agent

Stops the notebook's computation and releases its agent. The operation is idempotent: it also succeeds when the notebook has no active session. The
next cell execution starts a new agent and kernel with no variables or imports from the stopped process.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |

**Response:** `204 No Content`

---

## `POST /definitions` — Go to definition

Finds where a symbol is defined. **Requires a running kernel** — run any cell first.

**Request:**

| Field        | Type    | Required | Description                |
|--------------|---------|----------|----------------------------|
| `notebookId` | object  | yes      | Notebook identifier        |
| `cellId`     | string  | yes      | Cell containing the symbol |
| `line`       | integer | yes      | 1-based line number        |
| `column`     | integer | yes      | 1-based column number      |

**Response:**

```json
{"found": true, "cellId": "mP9wR2", "line": 1, "column": 1, "filePath": null}
```

When defined in an attached file: `cellId`/`line`/`column` are `null`, `filePath` has the path. When not found: `found` is `false`, all fields `null`.

---

## `POST /databases` — List attached databases

Lists all databases attached to the notebook.

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |

**Response:**

```json
{
  "databases": [
    {
      "databaseId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Production PostgreSQL",
      "driver": "PostgreSQL",
      "hasSchema": true
    }
  ]
}
```

If `hasSchema` is `false`, the database hasn't been introspected yet (user must do this from the Datalore UI).

---

## `POST /database/schema/children` — Browse schema tree

Navigates the schema tree and returns children at a given path.

**Request:**

| Field        | Type     | Required | Description                                         |
|--------------|----------|----------|-----------------------------------------------------|
| `notebookId` | object   | yes      | Notebook identifier                                 |
| `databaseId` | string   | yes      | Database UUID from `/databases`                     |
| `path`       | string[] | no       | Path to navigate into (default: `[]` = root)        |
| `depth`      | integer  | no       | Levels of children to include (default: 1, max: 10) |
| `verbosity`  | string   | no       | `essential` (default), `standard`, or `full`        |

**Schema tree structure:** Schema > Group > Object > Column. Groups are intermediate nodes like `"tables"`, `"views"`, `"routines"`. Path to a table:
`["public", "tables", "users"]`.

**Verbosity levels:**

- `essential`: TABLE, VIEW, SCHEMA, DATABASE, COLUMN, OTHER
- `standard`: + ROUTINE, FUNCTION, PROCEDURE, SEQUENCE, TYPE, SYNONYM
- `full`: all kinds (triggers, indexes, casts, etc.)

Group nodes (kind `OTHER`) and `kind: null` nodes are always included. Counts reflect the filtered tree.

**Response:**

```json
{
  "path": ["public", "tables"],
  "children": [
    {
      "name": "users",
      "kind": "TABLE",
      "info": "",
      "childrenCount": 3,
      "descendantCount": 3,
      "children": [
        {"name": "id", "kind": "column", "info": "serial, NOT NULL", "childrenCount": 0, "descendantCount": 0, "children": []},
        {"name": "name", "kind": "column", "info": "varchar(255)", "childrenCount": 0, "descendantCount": 0, "children": []}
      ]
    }
  ],
  "errors": null
}
```

`errors` is `null` when OK, a string message on partial introspection failures.

**Strategy based on `descendantCount`:**

- **< 100**: fetch full subtree with `depth: 5` in one call
- **Large**: navigate incrementally with `depth: 1`, or use schema search

---

## `POST /database/schema/condensed` — Get condensed schema

Returns the whole database schema as a compact text string intended for LLM/runtime-context use.

**Request:**

| Field        | Type   | Required | Description                     |
|--------------|--------|----------|---------------------------------|
| `notebookId` | object | yes      | Notebook identifier             |
| `databaseId` | string | yes      | Database UUID from `/databases` |

**Response:**

```json
{
  "schema": "Single-character type categories:\ni:integer,s:string,n:numeric,t:temporal,d:date,b:boolean,j:json,u:uid,y:binary,e:enum,g:geo\n\nDB: sales_prod\n SCHEMA: public\n  orders(id:i*,cust_id:i>customers.id,status:s,total:n)"
}
```

The `schema` string starts with the single-character type legend, then lists databases, schemas, tables, columns, primary keys, and foreign keys.
Primary key columns are marked with `*`; foreign key references use `>target_table.target_column` or a qualified target such as `>geo.regions.id`.
Type markers, primary keys, and references are based on database introspection metadata.

---

## `POST /database/schema/search` — Search schema

Searches for schema objects by name across the entire tree.

**Request:**

| Field        | Type     | Required | Description                                                  |
|--------------|----------|----------|--------------------------------------------------------------|
| `notebookId` | object   | yes      | Notebook identifier                                          |
| `databaseId` | string   | yes      | Database UUID                                                |
| `query`      | string   | yes      | Case-insensitive substring match on node names               |
| `kinds`      | string[] | no       | Filter by kind (e.g. `["TABLE", "VIEW"]`). Omit to match all |
| `limit`      | integer  | no       | Max results (default: 50, max: 200)                          |
| `verbosity`  | string   | no       | `essential` (default), `standard`, or `full`                 |

**Response:**

```json
{
  "results": [
    {
      "name": "users",
      "kind": "TABLE",
      "info": "",
      "path": ["public", "tables", "users"],
      "childrenCount": 5,
      "descendantCount": 5
    }
  ],
  "totalMatches": 1,
  "truncated": false
}
```

---

## `POST /files` — List files

Lists files in a directory attached to the notebook.

**Request:**

| Field        | Type   | Required | Description                                                    |
|--------------|--------|----------|----------------------------------------------------------------|
| `notebookId` | object | yes      | Notebook identifier                                            |
| `directory`  | string | no       | Directory path (default: root). Example: `data/notebook_files` |

**Response:**

```json
{
  "files": [
    {"name": "data.csv", "type": "FILE", "fileSize": 24576, "lastModified": 1709049600000},
    {"name": "models", "type": "FOLDER", "fileSize": 0, "lastModified": null}
  ]
}
```

File types: `FILE`, `FOLDER`, `PRIVATE_FOLDER`, `PLACEHOLDER_PRIVATE_FOLDER`

**File paths:** relative to notebook data root:

- `data/notebook_files` — notebook files (also cwd for code cells, so `open('myfile.csv')` reads from here)
- `data/workspace_files` — workspace-level files

---

## `POST /file` — Download file

Downloads a file as binary (`application/octet-stream`).

**Request:**

| Field        | Type   | Required | Description                                    |
|--------------|--------|----------|------------------------------------------------|
| `notebookId` | object | yes      | Notebook identifier                            |
| `filePath`   | string | yes      | Full path, e.g. `data/notebook_files/data.csv` |

**Response:** Binary file content with `Content-Disposition: attachment` header.

---

## `POST /file/upload` — Upload file

Uploads a file. Uses **multipart form data with query parameters** (not a JSON body).

**Query parameters:**

| Param        | Required | Description                      |
|--------------|----------|----------------------------------|
| `ownerId`    | yes      | Owner ID from notebook ID        |
| `notebookId` | yes      | Notebook internal ID             |
| `directory`  | no       | Target directory (default: root) |

**Multipart field:** `file` — the file to upload.

```python
import os
import requests

with open("local.csv", "rb") as f:
    response = requests.post(
        "https://<host>/api/notebook/v1/file/upload",
        headers={"Authorization": f"Bearer {os.environ['DATALORE_API_TOKEN']}"},
        params={"ownerId": "...", "notebookId": "...", "directory": "data/notebook_files"},
        files={"file": ("local.csv", f)},
    )
response.raise_for_status()
```

**Response:** `{"fileName": "local.csv", "status": "uploaded"}`

---

## `POST /file/delete` — Delete file

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |
| `filePath`   | string | yes      | Full path to file   |

**Response:** `204 No Content`

---

## `POST /file/exists` — Check file exists

**Request:**

| Field        | Type   | Required | Description         |
|--------------|--------|----------|---------------------|
| `notebookId` | object | yes      | Notebook identifier |
| `filePath`   | string | yes      | Full path to file   |

**Response:** `{"exists": true}`

---

## `POST /file/read` — Read file as text

Reads a file and returns its content as a decoded string. Works without a running kernel — use this instead of `open(path).read()` in a code cell when
you don't need the kernel up.

**Request:**

| Field        | Type    | Required | Description                                                                  |
|--------------|---------|----------|------------------------------------------------------------------------------|
| `notebookId` | object  | yes      | Notebook identifier                                                          |
| `filePath`   | string  | yes      | Full path, e.g. `data/notebook_files/config.txt`                             |
| `encoding`   | string  | no       | `utf-8` (default) or `latin-1`. Any other value → 400                        |
| `maxBytes`   | integer | no       | Byte cap. Default 1,048,576 (1 MiB), max 10,485,760 (10 MiB). Exceeded → 400 |
| `maxLines`   | integer | no       | Optional line cap. Must be positive if set                                   |

**Response:**

```json
{
  "content": "key=value\nother=42\n",
  "truncated": false,
  "bytesRead": 18,
  "linesRead": 2,
  "encoding": "utf-8"
}
```

`truncated` is `true` when either `maxBytes` or `maxLines` was hit before EOF — the content is exactly the prefix that fits.

**400** on: unsupported `encoding`, `maxBytes` out of range, `maxLines <= 0`, or decode failure (e.g., latin-1 bytes read as utf-8). **404** when the
file doesn't exist.

---

## Error handling

| HTTP Status | Meaning                                                                                                                                                                      |
|-------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 200         | Success                                                                                                                                                                      |
| 204         | Success, no content (delete cell, delete file)                                                                                                                               |
| 400         | Bad request (invalid cell type, unknown language, control type mismatch, `data` field on non-SQL cell, non-runnable cell in batch, file decode failure, out-of-range limits) |
| 401         | Missing or invalid API token                                                                                                                                                 |
| 403         | No access to the notebook                                                                                                                                                    |
| 404         | Cell/database/schema path/file not found                                                                                                                                     |
| 409         | Kernel-state precondition failed (e.g. `/kernel/interrupt` with no session). Body includes `state` with the current kernel state                                             |

Error body: `{"error": "message"}`. The 409 body additionally includes `"state": "NOT_STARTED" | "READY" | "BUSY"`.
