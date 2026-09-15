# Datalore Workspace API Reference

This document covers the current user/workspace-oriented public APIs used by the repository-local
`tools/skills/datalore-notebook/scripts/datalore` CLI.

## Base URLs

- User API: `https://<host>/api/user_public/v1`
- Workspace API: `https://<host>/api/workspaces/v1`

Derive `<host>` from any notebook or workspace URL in the same Datalore deployment.

## Authorization scopes

- `USER_BASIC`
  Required for `GET /api/user_public/v1/workspaces`. Any token whose scope covers `USER_BASIC` may call this endpoint, including `USER_BASIC`,
  `NOTEBOOK_API`, and `WORKSPACE`.
  The response always lists the caller's own workspaces; the token resource filter does not change the result set.

- `WORKSPACE`
  Required for Workspace API endpoints. `USER_FULL` also covers this scope. The required auth access depends on the endpoint: `READ` for reads,
  `CREATE` for folder/notebook creation, and `FULL` for destructive operations and database-query execution.

Every Workspace API endpoint requires an explicit `workspaceId=<ownerId>/<workspaceId>` query parameter. The server does not infer the target
workspace from the token resource filter; normal scope, auth-access, and workspace-access checks still apply.

## Identifier formats

- `workspaceId` query/auth values use the string form `<ownerId>/<workspaceId>`.
- `folderId` query values use the string form `<ownerId>/<folderId>`.
- `notebookId` query values use the string form `<ownerId>/<notebookId>`.

In JSON responses, IDs are returned as structured objects rather than slash-joined strings.

---

## `GET /workspaces` — List own workspaces

**Base URL:** `https://<host>/api/user_public/v1`

Returns the caller's own workspaces.

**Authorization**

- `USER_BASIC`
- any token whose scope covers `USER_BASIC` (`USER_BASIC`, `NOTEBOOK_API`, `WORKSPACE`, `USER_FULL`, `ADMIN_FULL`)
- resource filters do not narrow this response

**Response**

Returns a JSON array of workspace objects.

Top-level fields:

| Field              | Type             | Description                                         |
|--------------------|------------------|-----------------------------------------------------|
| `workspace`        | object           | Workspace properties                                |
| `owner`            | object           | Serialized `UserPublicInfo` for the workspace owner |
| `accessLevel`      | string           | Effective access level for the caller               |
| `accessorId`       | string or `null` | Caller user ID                                      |
| `isSharedWithTeam` | boolean          | Whether the workspace is team-shared                |
| `isCreatedFromGit` | boolean          | Whether the workspace is a Git workspace            |

`workspace` fields:

| Field               | Type          | Description                                   |
|---------------------|---------------|-----------------------------------------------|
| `id`                | object        | Workspace ID                                  |
| `homeFolderId`      | object        | Home folder ID                                |
| `trashFolderId`     | object        | Trash folder ID                               |
| `reportsFolderId`   | object        | Reports folder ID                             |
| `name`              | string        | Workspace name                                |
| `color`             | object/string | Serialized workspace color                    |
| `isDefault`         | boolean       | Whether this is the private/default workspace |
| `publicAccessLevel` | string        | Public access level                           |
| `isEmpty`           | boolean       | Whether the workspace is empty                |

Illustrative response:

```json
[
  {
    "workspace": {
      "id": {"ownerId": "alice", "internalId": "ws123"},
      "homeFolderId": {"ownerId": "alice", "id": "home123"},
      "trashFolderId": {"ownerId": "alice", "id": "trash123"},
      "reportsFolderId": {"ownerId": "alice", "id": "reports123"},
      "name": "Analysis",
      "isDefault": false,
      "publicAccessLevel": "NONE",
      "isEmpty": false
    },
    "owner": {
      "id": "alice",
      "name": "Alice"
    },
    "accessLevel": "OWNER",
    "accessorId": "alice",
    "isSharedWithTeam": false,
    "isCreatedFromGit": true
  }
]
```

---

## `GET /entries` — List direct entries in the workspace home folder

**Base URL:** `https://<host>/api/workspaces/v1`

Returns only the direct children of the explicitly selected workspace's home folder.

**Authorization**

- `WORKSPACE` with at least `READ` auth access
- `USER_FULL` tokens are also supported

**Query parameters**

| Param         | Required | Description                                    |
|---------------|----------|------------------------------------------------|
| `workspaceId` | yes      | Workspace ID in `<ownerId>/<workspaceId>` form |

**Response**

Returns a JSON array of workspace entry objects.

Current direct entry types exposed by this API are folders and notebooks.

Common top-level fields:

| Field | Type | Description |
|-------|------|-------------|
| `entity` | object | Serialized entity payload |
| `favorite` | boolean | Favorite marker |

For folder entries, the response also includes:

| Field | Type | Description |
|-------|------|-------------|
| `accessLevel` | string | Effective folder access |

For notebook entries, the response also includes:

| Field | Type | Description |
|-------|------|-------------|
| `personalAccessLevel` | string | Effective notebook access |
| `reportId` | object or `null` | Published report ID, if any |

Folder entity fields:

| Field              | Type             | Description      |
|--------------------|------------------|------------------|
| `entityId`         | object           | Entity ID        |
| `parentId`         | object or `null` | Parent folder ID |
| `workspaceId`      | object           | Workspace ID     |
| `name`             | string           | Folder name      |
| `modificationTime` | integer          | Unix millis      |
| `createdBy`        | string or `null` | Creator user ID  |

Notebook entity fields:

| Field                           | Type             | Description          |
|---------------------------------|------------------|----------------------|
| `id`                            | object           | Entity ID            |
| `parentId`                      | object           | Parent folder ID     |
| `workspaceId`                   | object           | Workspace ID         |
| `name`                          | string           | Notebook name        |
| `modificationTime`              | integer          | Unix millis          |
| `kernelType`                    | string           | Computation mode     |
| `instanceTypeId`                | string or `null` | Instance type        |
| `instanceStartupOptions`        | object or `null` | Startup options      |
| `runInBackgroundTimeoutSeconds` | integer          | Background timeout   |
| `language`                      | string           | Notebook language    |
| `languageVersion`               | string or `null` | Language version     |
| `kernelName`                    | string           | Kernel name          |
| `publicAccess`                  | string           | Public access level  |
| `createdBy`                     | string or `null` | Creator user ID      |
| `commentsEnabled`               | boolean          | Comments toggle      |
| `isSample`                      | boolean          | Sample notebook flag |

---

## `GET /reports?workspaceId=<ownerId>/<workspaceId>` — List all reports in the workspace

**Base URL:** `https://<host>/api/workspaces/v1`

Returns all published reports (both static and interactive) belonging to the specified workspace.

**Authorization**

- `WORKSPACE` with at least `READ` auth access
- `USER_FULL` tokens are also supported

**Query parameters**

| Param         | Required | Description                                    |
|---------------|----------|------------------------------------------------|
| `workspaceId` | yes      | Workspace ID in `<ownerId>/<workspaceId>` form |

**Response**

Returns a JSON array of report entity view models. Each report includes standard entity fields (id, name, modificationTime, etc.) along with
report-specific fields such as `fullWidth`, `reportFormat`, `isPublic`, and `publishedFrom`.

---

## `GET /folder-entries?workspaceId=<ownerId>/<workspaceId>&folderId=<ownerId>/<folderId>` — List direct entries in a folder

**Base URL:** `https://<host>/api/workspaces/v1`

Returns only the direct children of the specified folder.

**Authorization**

- `WORKSPACE` with at least `READ` auth access
- `USER_FULL` tokens are also supported

**Query parameters**

| Param         | Required | Description                                    |
|---------------|----------|------------------------------------------------|
| `workspaceId` | yes      | Workspace ID in `<ownerId>/<workspaceId>` form |
| `folderId`    | yes      | Folder ID in `<ownerId>/<folderId>` form       |

**Behavior**

- The folder must exist.
- The folder must belong to the explicitly selected workspace.
- The caller must have read access to the folder and workspace.

**Response**

Same workspace-entry shape as `GET /entries`.

---

## `POST /folder` — Create a folder

**Base URL:** `https://<host>/api/workspaces/v1`

Creates a new folder in the specified parent folder (or the workspace home folder if `parentId` is omitted).

**Authorization**

- `WORKSPACE` with at least `CREATE` auth access
- `USER_FULL` tokens are also supported

**Query parameters**

| Param         | Required | Description                                                                        |
|---------------|----------|------------------------------------------------------------------------------------|
| `workspaceId` | yes      | Workspace ID in `<ownerId>/<workspaceId>` form                                     |
| `name`        | yes      | Folder name                                                                        |
| `parentId`    | no       | Parent folder ID in `<ownerId>/<folderId>` form; defaults to workspace home folder |

**Response**

Returns a JSON entity ID object for the created folder.

---

## `POST /notebook` — Create a notebook

**Base URL:** `https://<host>/api/workspaces/v1`

Creates a new notebook in the specified parent folder (or the workspace home folder if `parentId` is omitted).

**Authorization**

- `WORKSPACE` with at least `CREATE` auth access
- `USER_FULL` tokens are also supported

**Query parameters**

| Param         | Required | Description                                                                        |
|---------------|----------|------------------------------------------------------------------------------------|
| `workspaceId` | yes      | Workspace ID in `<ownerId>/<workspaceId>` form                                     |
| `name`        | yes      | Notebook name                                                                      |
| `language`    | yes      | Notebook language (e.g., `PYTHON`)                                                 |
| `parentId`    | no       | Parent folder ID in `<ownerId>/<folderId>` form; defaults to workspace home folder |

**Response**

Returns a JSON entity ID object for the created notebook.

---

## `POST /database/query` — Execute a workspace database query

**Base URL:** `https://<host>/api/workspaces/v1`

Creates a database query using the supplied name and SQL text, then executes it immediately. The query content type is fixed to `text/plain` and the
result format is fixed to CSV.

**Authorization**

- `WORKSPACE` with `FULL` auth access
- `USER_FULL` tokens are also supported
- the caller must have write access to the selected workspace

**Query parameters**

| Param         | Required | Description                                    |
|---------------|----------|------------------------------------------------|
| `workspaceId` | yes      | Workspace ID in `<ownerId>/<workspaceId>` form |
| `databaseId`  | yes      | ID of a database scoped to that workspace      |
| `name`        | yes      | Name stored for the created database query     |

**Request body**

Send the SQL query as a raw string with `Content-Type: text/plain`.

```http
POST /api/workspaces/v1/database/query?workspaceId=alice/ws123&databaseId=db123&name=Recent%20orders
Content-Type: text/plain
Accept: text/csv

SELECT * FROM orders LIMIT 10
```

**Response**

Returns the query result as `text/csv`. It is streamed directly rather than wrapped in JSON.

**Behavior**

- Returns `404` when the database does not exist in the requested workspace.
- Creates a query record under the caller before executing it.
- Executes with `content_type=text/plain` and `response_content_type=csv`.

---

## Error handling

| HTTP Status | Meaning                                                                       |
|-------------|-------------------------------------------------------------------------------|
| 200         | Success                                                                       |
| 400         | Malformed workspace query parameter such as `folderId`                        |
| 401         | Missing or invalid bearer token                                               |
| 403         | Valid token, but insufficient scope or access                                 |
| 404         | Workspace/folder/entity not found, or entity is outside the token's workspace |

For scope failures, the server may include:

```http
WWW-Authenticate: Bearer scope="WORKSPACE", access="FULL"
```

This is the value used by the repository-local CLI to suggest a new authorization flow with the required scope.
