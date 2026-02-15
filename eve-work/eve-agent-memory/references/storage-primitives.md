# Eve Horizon Storage Primitives Reference

Complete inventory of storage systems available to agents on Eve Horizon, with CLI commands, API endpoints, and design considerations.

## 1. Workspace Files

**Scope**: Job
**Lifetime**: Job duration (unless committed to git)
**Access**: Agent running the job

The git repo checkout at `$EVE_REPO_PATH`. Workspace mode controls sharing:

| Mode | Behavior |
|------|----------|
| `job` | Fresh checkout per job (default) |
| `session` | Shared across jobs using the same `workspace-key` |
| `isolated` | No git state, pure scratch space |

```bash
eve job create --workspace-mode session --workspace-key "my-session"
```

Convention: write agent scratch to `.eve/` (gitignored). Committed files persist in the repo.

## 2. Job Attachments

**Scope**: Job (readable by any job with the ID)
**Lifetime**: Permanent (attached to job record)
**Access**: Any agent with the job ID

Named binary blobs attached to a job record.

```bash
# Write
eve job attach <job-id> --name <name> [--content <text>] [--file <path>] [--content-type <type>]

# Read
eve job attachments <job-id>                    # list metadata
eve job attachment <job-id> <name> [--out <path>]  # retrieve content
```

API:
```
POST /jobs/:id/attachments
GET  /jobs/:id/attachments
GET  /jobs/:id/attachments/:attachment_id
```

## 3. Threads

**Scope**: Project or Org
**Lifetime**: Permanent
**Access**: RBAC (user tokens) or job tokens (project-scoped)

Message sequences for continuity and coordination.

### Project Threads
```bash
eve thread messages <thread-id> --since 10m
eve thread post <thread-id> --body '{"kind":"update","body":"progress note"}'
eve thread follow <thread-id>     # poll every ~3s
```

### Org Threads
```bash
eve thread list --org <org-id>
eve thread show --org <org-id> --id <thread-id>
eve thread messages <thread-id> --org <org-id> --since 10m
eve thread post <thread-id> --body '{"kind":"directive","body":"instructions"}'
```

### Coordination Threads
Auto-created for team dispatches. Key: `coord:job:{parent_job_id}`.

Child agents derive the key from `$EVE_PARENT_JOB_ID`. End-of-attempt summaries are posted automatically. The coordination inbox (`.eve/coordination-inbox.md`) is regenerated at job start.

### Message Kinds
| Kind | Purpose |
|------|---------|
| `status` | Automatic end-of-attempt summary |
| `directive` | Lead-to-member instruction |
| `question` | Member-to-lead question |
| `update` | Progress update from member |

### Listener Subscriptions
```
@eve agents listen <agent-slug>    # subscribe agent to channel/thread
@eve agents unlisten <agent-slug>
@eve agents listening              # list active listeners
```

## 4. Resource Refs

**Scope**: Job
**Lifetime**: Job duration (references resolved at hydration)
**Access**: Agent running the job

Versioned pointers to org documents, mounted into job workspaces.

```bash
eve job create \
  --resource-refs='[{"uri":"org_docs:/path/to/doc.md@v3","label":"Input","mount_path":"inputs/doc.md"}]'
```

URI scheme: `org_docs:/path@version`. Hydration emits events:
- `system.resource.hydration.started`
- `system.resource.hydration.completed`
- `system.resource.hydration.failed`

## 5. Org Document Store

**Scope**: Org
**Lifetime**: Permanent, versioned
**Access**: Org membership (+ access groups for fine-grained control)

Versioned, searchable document store scoped to the organization.

```bash
# CRUD
eve docs create --org <org-id> --path /path/to/doc.md --file ./content.md
eve docs get --org <org-id> --path /path/to/doc.md
eve docs update --org <org-id> --path /path/to/doc.md --file ./updated.md
eve docs delete --org <org-id> --path /path/to/doc.md

# List and search
eve docs list --org <org-id> --prefix /agents/
eve docs search --org <org-id> --query "keyword"

# Version history
eve docs versions --org <org-id> --path /path/to/doc.md
```

Events emitted: `system.doc.created`, `system.doc.updated`, `system.doc.deleted`.

API:
```
POST   /orgs/:org_id/docs
GET    /orgs/:org_id/docs
GET    /orgs/:org_id/docs/:doc_id
PUT    /orgs/:org_id/docs/:doc_id
DELETE /orgs/:org_id/docs/:doc_id
GET    /orgs/:org_id/docs/search?q=<query>
```

## 6. Org Filesystem (Sync)

**Scope**: Org
**Lifetime**: Permanent
**Access**: Org permissions (orgfs scope with access groups)

Bidirectional file sync between local machines and org storage. Syncthing data plane with control-plane APIs.

```bash
# Setup
eve fs sync init --org <org-id> --local ~/Eve/acme --mode two-way
  # Modes: two-way | push-only | pull-only
  # Optional: --include '**/*.md' --exclude '**/node_modules/**'

# Operations
eve fs sync status --org <org-id>
eve fs sync logs --org <org-id> --follow
eve fs sync pause --org <org-id>
eve fs sync resume --org <org-id>
eve fs sync disconnect --org <org-id>
eve fs sync mode --org <org-id> --set pull-only

# Conflict resolution
eve fs sync conflicts --org <org-id>
eve fs sync resolve --org <org-id> --conflict <id> --strategy pick-remote

# Diagnostics
eve fs sync doctor --org <org-id>
```

Event types: `file.created`, `file.updated`, `file.deleted`, `file.renamed`, `conflict.detected`, `conflict.resolved`.

SSE stream: `GET /orgs/:org_id/fs/events/stream?after_seq=<n>`

Default includes: `**/*.md`, `**/*.mdx`, `**/*.txt`, `**/*.yaml`, `**/*.yml`.

## 7. Managed Databases

**Scope**: Environment
**Lifetime**: Permanent (until environment destroyed)
**Access**: Workers via injected credentials; CLI via `eve db`

Postgres instances declared in the manifest and provisioned per environment.

```yaml
# eve.yaml
services:
  db:
    x-eve:
      role: managed_db
      managed:
        class: db.p1
        engine: postgres
        engine_version: "16"
```

```bash
# Query
eve db sql --env <env> --sql "SELECT * FROM agent_memory WHERE key = 'patterns'"
eve db sql --env <env> --sql "INSERT INTO ..." --write

# Schema
eve db schema --env <env>
eve db migrate --env <env>
eve db new create_agent_memory

# RLS (row-level security with group awareness)
eve db rls --env <env>
eve db rls init --with-groups  # scaffold group-aware policies
```

## 8. Secrets

**Scope**: System / Org / User / Project
**Lifetime**: Until rotated or deleted
**Access**: Resolution hierarchy (project → user → org → system)

Encrypted credential storage. Not for agent memory — for API keys, tokens, and credentials.

```bash
eve secrets set MY_KEY myvalue --scope project --project <id>
eve secrets show MY_KEY --project <id>
eve secrets import --org <org-id> --file ./secrets.env
eve secrets validate --project <id>
```

## 9. Event Spine

**Scope**: Project or Org
**Lifetime**: Permanent (event records)
**Access**: RBAC

Pub/sub event bus for reactive workflows. Not a storage primitive per se, but critical for memory-aware architectures.

```bash
eve event emit --type=<type> --source=<source> --payload '<json>'
eve event list [--type <type>] [--source <source>] [--status <status>]
eve event show <event-id>
```

Event types relevant to memory:
- `system.doc.created/updated/deleted` — org document mutations
- `system.resource.hydration.*` — resource ref resolution
- `runner.*` — job lifecycle events
- Custom `agent.memory.*` types for agent-specific notifications

## 10. Skills and Skillpacks

**Scope**: Org or global (via skillpack repos)
**Lifetime**: Permanent, version-controlled
**Access**: Skill installation and loading

The highest-fidelity memory: distilled workflows encoded as agent instructions. See `eve-skill-distillation` for creation workflow.

```bash
eve skill install --from ../eve-skillpacks/eve-work/
eve skill list
```

## 11. Container Registry

**Scope**: Project
**Lifetime**: Permanent (image tags)
**Access**: Project membership

Store and retrieve container images. Not directly a memory primitive, but relevant for agents that build and deploy.

```bash
eve registry push --tag <tag>
eve registry images
eve registry tags <image>
```

## Comparison Matrix

| Primitive | Scope | Searchable | Versioned | Structured | Event-Driven |
|-----------|-------|------------|-----------|------------|-------------|
| Workspace | Job | No | Via git | Files | No |
| Attachments | Job | By name | No | Key-value | No |
| Threads | Project/Org | By time | No | Messages | No |
| Resource Refs | Job | No | Yes (pinned) | URI | Yes |
| Org Docs | Org | Yes (text) | Yes | Markdown | Yes |
| Org Filesystem | Org | No* | Via events | Files | Yes |
| Managed DB | Environment | Yes (SQL) | No | Tables | No |
| Secrets | Multi-scope | By key | No | Key-value | No |
| Event Spine | Project/Org | By type/time | No | JSON | Yes (core) |
| Skills | Global | No | Via git | Markdown | No |

*Org Filesystem search requires local tooling or combining with org docs.
