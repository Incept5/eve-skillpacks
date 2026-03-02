# Object Store & Org Filesystem Reference

Unified storage layer for Eve Horizon: S3-compatible object storage backing org filesystem sync and app object buckets.

## Use When

- You need to set up or troubleshoot org filesystem sync between local machines and Eve.
- You need to share files via share tokens or public paths.
- You need to understand how agents access the org filesystem at runtime.
- You need to declare object store buckets for an app service.

## Load Next

- `references/cli-org-project.md` for org/project setup and docs CLI commands.
- `references/secrets-auth.md` for access groups and scoped bindings (orgfs permissions).
- `references/events.md` for event-driven automation triggered by file changes.

## Ask If Missing

- Confirm target org ID before running any `eve fs` command.
- Confirm sync mode needed (two-way, push-only, pull-only) before initializing.
- Confirm include/exclude patterns if syncing a subset of files.

## Overview

Three storage primitives share a common object store backend:

| Primitive | Scope | Use Case |
|-----------|-------|----------|
| **Object Store** | Platform | Binary file storage via presigned URL transfers |
| **Org Filesystem** | Org | Multi-device file sync with real-time events |
| **Org Docs** | Org | Versioned document store with full-text search |

The object store is MinIO locally (k3d dev) and S3/GCS/R2/Tigris in cloud. All speak the S3 protocol.

## Org Filesystem

### Sync Protocol

Files transfer via **presigned URLs** -- content never flows through the Eve API:

- **Upload**: CLI detects change -> computes SHA-256 -> gets presigned PUT URL -> uploads direct to S3
- **Download**: SSE event stream delivers presigned GET URL -> CLI downloads direct from S3

### Sync Modes

| Mode | Behavior |
|------|----------|
| `two-way` | Bidirectional sync (default) |
| `push-only` | Local -> remote only |
| `pull-only` | Remote -> local only |

### CLI Commands

```bash
# Initialize sync
eve fs sync init --org <org> --local <path> [--mode two-way|push-only|pull-only] \
  [--remote-path /] [--include "**/*.md"] [--exclude "**/.git/**"]

# Status and monitoring
eve fs sync status --org <org>
eve fs sync logs --org <org> [--follow]
eve fs sync doctor --org <org>

# Link management
eve fs sync pause --org <org>
eve fs sync resume --org <org>
eve fs sync disconnect --org <org>
eve fs sync mode --org <org> --set <mode>

# Conflict resolution
eve fs sync conflicts --org <org>
eve fs sync resolve --org <org> --conflict <id> --strategy <pick-remote|pick-local|manual>
```

### Share Tokens

Time-limited, revocable access to individual files:

```bash
eve fs share <path> --org <org> [--expires 7d] [--label "description"]
eve fs shares --org <org>
eve fs revoke <token> --org <org>
```

### Public Paths

Permanent unauthenticated access to path prefixes:

```bash
eve fs publish <path-prefix> --org <org> [--label "description"]
eve fs public-paths --org <org>
```

Public file resolver (no auth): `GET /orgs/{orgId}/fs/public/{path}`

### Text Indexing

Text files (markdown, YAML, JSON; under 512 KB) synced to the org filesystem are automatically indexed into org documents for full-text search. Indexing is async (poll interval: 2s, batch: 10).

### Events

| Event | Trigger |
|-------|---------|
| `file.created` | New file uploaded |
| `file.updated` | Existing file modified |
| `file.deleted` | File removed |
| `conflict.detected` | Both sides modified same file |

SSE stream: `GET /orgs/{orgId}/fs/events/stream?after_seq=<n>`

### Agent Runtime

Warm pods mount org filesystem as PVC at `/org` (`EVE_ORG_FS_ROOT=/org`). Agents read/write directly -- changes sync to S3 and index into org docs automatically.

## App Object Stores

> **Status: Schema exists, provisioning logic pending.** The `storage_buckets` table and bucket naming convention are implemented. App-level bucket provisioning from the manifest is not yet wired.

Manifest declaration (planned):

```yaml
services:
  api:
    x-eve:
      object_store:
        buckets:
          - name: uploads
            visibility: private
          - name: avatars
            visibility: public
            cors:
              allowed_origins: ["*"]
```

Each bucket provisioned per environment. Auto-injected env vars:

| Variable | Description |
|----------|-------------|
| `STORAGE_ENDPOINT` | S3-compatible endpoint |
| `STORAGE_ACCESS_KEY` | Access key |
| `STORAGE_SECRET_KEY` | Secret key |
| `STORAGE_BUCKET` | Physical bucket name |
| `STORAGE_FORCE_PATH_STYLE` | `true` for MinIO, `false` for cloud |

## Access Control

| Permission | Allows |
|------------|--------|
| `orgfs:read` | List, download, view shares and public paths |
| `orgfs:write` | Upload, create links, resolve conflicts |
| `orgfs:admin` | Manage share tokens, publish/unpublish public paths |

Links support path-scoped ACLs via `scope_json.allow_prefixes`.

## Org Docs (Versioned Document Store)

Covered in detail in the `storage-primitives.md` reference (Section 5). Key points relevant to the storage layer:

- Org docs is Postgres-native -- content stored in rows, not the object store
- Full-text search via `tsvector` with weighted path (A) and content (B)
- Async indexer bridges org filesystem -> org docs for text files
- Search modes: `text` (default), `semantic`, `hybrid` (semantic/hybrid degrade to text when embeddings absent)
- Lifecycle: `--review-in`, `--expires-in`, `eve docs stale`, `eve docs review`
