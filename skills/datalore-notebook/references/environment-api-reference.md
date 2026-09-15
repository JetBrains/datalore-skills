# Environment Resource API Reference

## Overview

Environment resources represent collections of environment variables in Datalore workspaces. They provide a way to configure environment variables for
notebooks and computations.

## API Endpoints

All endpoints are under `/api/environments/v1/`.

### Resource Endpoints

All operations use `POST` method with JSON body.

| Endpoint      | Body                                         | Description                        |
|---------------|----------------------------------------------|------------------------------------|
| `/create`     | Object with `scopeId` and `resource`         | Create a new environment resource  |
| `/get`        | Environment ID (UUID string)                 | Get a specific environment by ID   |
| `/update`     | Environment JSON object                      | Update an existing environment     |
| `/delete`     | Environment ID (UUID string)                 | Delete an environment              |
| `/all`        | Workspace scope object                       | List all environments in scope     |
| `/clone`      | `scopes` object + `resource_id` parameter    | Clone an environment to new scopes |
| `/search`     | Object with `query` and `workspaces`         | Search environments by name        |
| `/usage-list` | Environment ID + `matchDuplicates` parameter | List where an environment is used  |

## Environment Schema

```json
{
  "id": "string (UUID)",
  "name": "string",
  "variables": [
    {
      "key": "string",
      "value": "string"
    }
  ]
}
```

## Field Notes

- The `variables` array contains key-value pairs of environment variables.
- Like all resources, environments are scoped to workspaces.
