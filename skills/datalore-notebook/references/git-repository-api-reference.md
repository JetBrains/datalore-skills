# Git Repository Resource API Reference

## Overview

Git repository resources represent Git repository connections in Datalore workspaces. They can use SSH keys for authentication.

## API Endpoints

All endpoints are under `/api/git-repositories/v1/`.

### Resource Endpoints

All resource operations use `POST` with JSON body. The base path is `/api/git-repositories/v1/`.

| Endpoint      | Body                                            | Description                          |
|---------------|-------------------------------------------------|--------------------------------------|
| `/create`     | Object with `scopeId` and `resource`            | Create a new Git repository resource |
| `/get`        | Git repository ID (UUID string)                 | Get a specific Git repository by ID  |
| `/update`     | Git repository JSON object                      | Update an existing Git repository    |
| `/delete`     | Git repository ID (UUID string)                 | Delete a Git repository              |
| `/all`        | Workspace scope object                          | List all Git repositories in scope   |
| `/clone`      | `scopes` object + `resource_id` parameter       | Clone a Git repository to new scopes |
| `/search`     | Object with `query` and `workspaces`            | Search Git repositories by name      |
| `/usage-list` | Git repository ID + `matchDuplicates` parameter | List where a Git repository is used  |

## Git Repository Schema

```json
{
  "id": "string (UUID)",
  "name": "string",
  "url": "string (Git URL, e.g. 'git+ssh://git@github.com/org/repo.git')",
  "sshKeyId": "string (UUID) | null (reference to SSH key resource)",
  "defaultReference": {
    "type": "Head | Branch | Tag | Commit | Unknown",
    "reference": "string (branch/tag name, commit SHA, or HEAD)"
  },
  "creationTime": "string (ISO 8601 timestamp)"
}
```

## Git URL Formats

Git URLs support multiple protocols:

| Protocol    | Format                             | Example                                 |
|-------------|------------------------------------|-----------------------------------------|
| SSH         | `git+ssh://git@host/path/repo.git` | `git+ssh://git@github.com/org/repo.git` |
| SSH (short) | `git@host:path/repo.git`           | `git@github.com:org/repo.git`           |
| HTTPS       | `git+https://host/path/repo.git`   | `git+https://github.com/org/repo.git`   |
| HTTP        | `git+http://host/path/repo.git`    | `git+http://github.com/org/repo.git`    |

## Git Reference Schema

```json
{
  "type": "Head | Branch | Tag | Commit | Unknown",
  "reference": "string"
}
```

- **Branch**: `reference` contains the branch name (for example, `"main"`)
- **Tag**: `reference` contains the tag name (for example, `"v1.0.0"`)
- **Commit**: `reference` contains the full commit SHA

## SSH Key Authentication

When `sshKeyId` is set, the referenced SSH key resource is used for authentication. This is required for private repositories accessed via SSH.
