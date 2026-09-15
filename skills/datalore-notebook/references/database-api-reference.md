# Database Resource API Reference

## Overview

Database resources represent database connections configured in Datalore workspaces. They support various authentication providers including
username/password and OAuth.

## API Endpoints

All endpoints are under `/api/databases/v1/`.

### Resource Endpoints

All resource operations use `POST` with JSON body. The base path is `/api/databases/v1/`.

| Endpoint      | Body                                      | Description                    |
|---------------|-------------------------------------------|--------------------------------|
| `/create`     | Object with `scopeId` and `resource`      | Create a new database resource |
| `/get`        | Database ID (UUID string)                 | Get a specific database by ID  |
| `/update`     | Database JSON object                      | Update an existing database    |
| `/delete`     | Database ID (UUID string)                 | Delete a database              |
| `/all`        | Workspace scope object                    | List all databases in scope    |
| `/clone`      | `scopes` object + `resource_id` parameter | Clone a database to new scopes |
| `/search`     | Object with `query` and `workspaces`      | Search databases by name       |
| `/usage-list` | Database ID + `matchDuplicates` parameter | List where a database is used  |

### Database Driver Endpoint

All use `POST` method.

| Endpoint           | Body    | Description                        |
|--------------------|---------|------------------------------------|
| `/get-all-drivers` | (empty) | Get all available database drivers |

## Database Schema

```json
{
  "id": "string (UUID)",
  "name": "string",
  "driverRef": {
    "is_predefined": boolean,
    "id": "string (driver ID, e.g. 'postgres', 'mysql')"
  },
  "userName": "string | null",
  "password": "string | null (hidden in GET responses)",
  "params": {
    "URL": "string (JDBC URL)",
    "host": "string",
    "port": "string",
    "...": "additional connection params"
  },
  "paramsGroupId": "integer",
  "additionalProperties": {
    "...": "auth provider specific properties"
  },
  "advancedProperties": {
    "...": "advanced configuration"
  },
  "keys": {
    "...": "key-value pairs (hidden in GET responses)"
  },
  "authProviderId": "string (default: 'userPass')",
  "sshTunnelConfiguration": {
    "host": "string",
    "port": "integer",
    "username": "string",
    "credentials": {
      "type": "PASSWORD | KEY_PAIR",
      "password": "string | null",
      "sshKeyId": "string (UUID) | null"
    }
  },
  "introspectionScope": {
    "name": "string",
    "info": "string",
    "obj": "string",
    "kind": "string",
    "isChecked": boolean,
    "children": ["..."]
  },
  "vmOptions": "string | null",
  "useExperimentalSession": "boolean (default: true)"
}
```

## Driver Schema

```json
{
  "id": "string (driver ID)",
  "name": "string (display name)",
  "featured": boolean,
  "iconPath": "string",
  "authProviders": ["string"],
  "urlModel": {
    "...": "URL model definition"
  },
  "predefined": boolean
}
```

## SSH Tunnel Configuration

Databases support SSH tunnel connections with two authentication types:

- **PASSWORD**: SSH password authentication
- **KEY_PAIR**: SSH key pair authentication (references an SshKey resource by ID)
