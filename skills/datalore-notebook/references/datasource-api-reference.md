# Datasource Resource API Reference

## Overview

Datasource resources represent generic data source connections in Datalore workspaces. They use a specification-based approach where different
datasource kinds define their own field schemas.

## API Endpoints

All endpoints are under `/api/datasources/v1/`.

### Resource Endpoints

All resource operations use `POST` with JSON body. The base path is `/api/datasources/v1/`.

| Endpoint      | Body                                        | Description                      |
|---------------|---------------------------------------------|----------------------------------|
| `/create`     | Object with `scopeId` and `resource`        | Create a new datasource resource |
| `/get`        | Datasource ID (UUID string)                 | Get a specific datasource by ID  |
| `/update`     | Datasource JSON object                      | Update an existing datasource    |
| `/delete`     | Datasource ID (UUID string)                 | Delete a datasource              |
| `/all`        | Workspace scope object                      | List all datasources in scope    |
| `/clone`      | `scopes` object + `resource_id` parameter   | Clone a datasource to new scopes |
| `/search`     | Object with `query` and `workspaces`        | Search datasources by name       |
| `/usage-list` | Datasource ID + `matchDuplicates` parameter | List where a datasource is used  |

## Datasource Schema

```json
{
  "id": "string (UUID)",
  "name": "string",
  "kind": "string (datasource type, e.g. 's3', 'gcs', 'azure-blob')",
  "fields": {
    "field_name_1": "string | null",
    "field_name_2": "string | null",
    "...": "additional fields as defined by the specification"
  }
}
```

## Field Notes

- Fields marked as `hidden` in datasource metadata have their values hidden in GET responses because they contain secrets.
- When creating or updating a datasource, provide all fields required by its kind.
- The `kind` field determines the accepted field set.
