# SSH Key Resource API Reference

## Overview

SSH key resources represent SSH key pairs in Datalore workspaces. They are used for authentication with Git repositories and SSH tunnel connections to
databases.

## API Endpoints

All endpoints are under `/api/ssh-keys/v1/`.

### Resource Endpoints

All resource operations use `POST` with JSON body. The base path is `/api/ssh-keys/v1/`.

| Endpoint      | Body                                      | Description                    |
|---------------|-------------------------------------------|--------------------------------|
| `/create`     | Object with `scopeId` and `resource`      | Create a new SSH key resource  |
| `/get`        | SSH key ID (UUID string)                  | Get a specific SSH key by ID   |
| `/update`     | SSH key JSON object                       | Update an existing SSH key     |
| `/delete`     | SSH key ID (UUID string)                  | Delete an SSH key              |
| `/all`        | Workspace scope object                    | List all SSH keys in scope     |
| `/clone`      | `scopes` object + `resource_id` parameter | Clone an SSH key to new scopes |
| `/search`     | Object with `query` and `workspaces`      | Search SSH keys by name        |
| `/usage-list` | SSH key ID + `matchDuplicates` parameter  | List where an SSH key is used  |

### SSH Key-Specific Endpoints

| Endpoint    | Body                     | Description                          |
|-------------|--------------------------|--------------------------------------|
| `/generate` | `algorithm` parameter    | Generate a new SSH key pair          |
| `/parse`    | Object with `privateKey` | Parse a supplied private key for use |

## SSH Key Schema

```json
{
  "id": "string (UUID)",
  "name": "string",
  "privateKey": "string (hidden in GET responses)",
  "publicKey": "string (SSH public key)",
  "md5Fingerprint": "string",
  "sha256Fingerprint": "string",
  "comment": "string | null",
  "creationTime": "string (ISO 8601 timestamp)"
}
```

## SSH Key Algorithms

The SSH key generation supports the following algorithms:

| Algorithm | Identifier |
|-----------|------------|
| RSA       | `RSA`      |
| ED25519   | `ED25519`  |

## Field Notes

- The `privateKey` field is always hidden in GET responses for security.
- Fingerprints (`md5Fingerprint`, `sha256Fingerprint`) are computed from the public key.
- SSH keys can be referenced by Git repositories (`sshKeyId` field) and database SSH tunnel configurations.
