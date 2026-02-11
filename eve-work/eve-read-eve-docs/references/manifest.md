# Manifest v2 (Current)

The manifest (`.eve/manifest.yaml`) is the single source of truth for builds, deploys, pipelines, and workflows.
Schema is Compose-like with Eve extensions under `x-eve`.

## Top-Level Fields

```yaml
schema: eve/compose/v2          # optional
project: my-project             # optional slug
registry:                        # optional container registry
services:                        # required
environments:                    # optional
pipelines:                       # optional
workflows:                       # optional
versioning:                      # optional
x-eve:                           # optional Eve extensions
```

Unknown fields are allowed for forward compatibility.

## Registry

```yaml
registry:
  host: ghcr.io
  namespace: myorg
  auth:
    username_secret: GHCR_USERNAME
    token_secret: GHCR_TOKEN
```

String modes:

```yaml
registry: "eve"   # Use Eve-native registry (internal)
registry: "none"  # Disable registry handling
```

## Services (Compose-Style)

```yaml
services:
  api:
    build:
      context: ./apps/api
    image: ghcr.io/org/api       # optional if build is provided
    ports: [3000]
    environment:
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
    x-eve:
      ingress:
        public: true
        port: 3000
      api_spec:
        type: openapi
        spec_url: /openapi.json
```

Supported Compose fields: `image`, `build`, `environment`, `ports`, `depends_on`, `healthcheck`, `volumes`.

### Eve Service Extensions (`x-eve`)

| Field | Type | Description |
|-------|------|-------------|
| `role` | string | `component` (default), `worker`, `job`, or `managed_db` |
| `ingress` | object | `{ public: true\|false, port: number }` |
| `api_spec` | object | Single API spec registration |
| `api_specs` | array | Multiple API spec registrations |
| `external` | boolean | External dependency (not deployed) |
| `connection_url` | string | Connection string for external services |
| `worker_type` | string | Worker pool type for this service |
| `files` | array | Mount source files into container |
| `storage` | object | Persistent volume configuration |
| `managed` | object | Managed DB config (requires `role: managed_db`) |

Notes:
- `x-eve.role: job` makes a service runnable as a one-off job (migrations, seeds).
- `x-eve.role: managed_db` marks a service as a platform-provisioned database.
- `spec_url` can be relative (resolved against service URL) or absolute.
- `spec_path` is supported only for local `file://` repos.

### Managed DB Services

```yaml
services:
  db:
    x-eve:
      role: managed_db
      managed:
        class: db.p1
        engine: postgres
        engine_version: "16"
```

### API Spec Schema

```yaml
api_spec:
  type: openapi              # openapi | postgrest | graphql
  spec_url: /openapi.json    # relative to service URL, or absolute
  spec_path: ./openapi.yaml  # local file path (file:// repos only)
  name: my-api               # optional display name
  auth: eve                  # eve (default) | none
  on_deploy: true            # refresh on deploy (default: true)
```

Multiple specs:

```yaml
api_specs:
  - type: openapi
    spec_url: /openapi.json
  - type: graphql
    spec_url: /graphql
```

### Files Mount

Mount source files from the repo into the container:

```yaml
x-eve:
  files:
    - source: ./config/app.conf    # relative path in repo
      target: /etc/app/app.conf    # absolute path in container
```

### Persistent Storage

```yaml
x-eve:
  storage:
    mount_path: /data
    size: 10Gi
    access_mode: ReadWriteOnce     # ReadWriteOnce | ReadWriteMany | ReadOnlyMany
    storage_class: standard        # optional
    name: my-data                  # optional PVC name
```

### Healthcheck

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  interval: 5s
  timeout: 3s
  retries: 3
  start_period: 10s
```

### Dependency Conditions

```yaml
depends_on:
  db:
    condition: service_healthy     # service_started | service_healthy | started | healthy
```

## Environments

```yaml
environments:
  staging:
    pipeline: deploy
    pipeline_inputs:
      smoke_test: true
    approval: required
    overrides:
      services:
        api:
          environment:
            NODE_ENV: staging
    workers:
      - type: default
        service: worker
        replicas: 2
```

- If `pipeline` is set, `eve env deploy <env>` triggers that pipeline.
- Use `--direct` to bypass pipeline and deploy directly.
- `overrides` is Compose-style and merges into services.

### Environment Fields

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `persistent` (default) or `temporary` |
| `kind` | string | `standard` (default) or `preview` (PR envs) |
| `pipeline` | string | Pipeline name to trigger on deploy |
| `pipeline_inputs` | object | Inputs passed to pipeline |
| `approval` | string | `required` to gate deploys |
| `overrides` | object | Compose-style service overrides |
| `workers` | array | Worker pool configuration |
| `labels` | object | Metadata (PR info for preview envs) |

## Pipelines (Steps)

```yaml
pipelines:
  deploy-test:
    steps:
      - name: migrate
        action: { type: job, service: migrate }
      - name: deploy
        depends_on: [migrate]
        action: { type: deploy }
```

Step types: `action`, `script`, `agent`, or shorthand `run`.

See `references/pipelines-workflows.md` for step types, triggers, and the canonical build-release-deploy pattern.

Platform env vars injected into services:
- `EVE_API_URL` — internal cluster URL for server-to-server calls
- `EVE_PUBLIC_API_URL` — public ingress URL for browser-facing apps
- `EVE_PROJECT_ID`, `EVE_ORG_ID`, `EVE_ENV_NAME`

## Workflows

```yaml
workflows:
  nightly-audit:
    db_access: read_only
    hints:
      gates: ["remediate:proj_xxx:staging"]
    steps:
      - agent:
          prompt: "Audit error logs"
```

Workflow invocation creates a job with the workflow hints merged.

## Manifest Defaults (`x-eve.defaults`)

Default job settings applied on creation (job fields override defaults). Default environment should be **staging** unless explicitly overridden:

```yaml
x-eve:
  defaults:
    env: staging
    harness: mclaude
    harness_profile: primary-orchestrator
    harness_options:
      model: opus-4.5
      reasoning_effort: high
    hints:
      permission_policy: auto_edit
    git:
      ref_policy: auto
      branch: job/${job_id}
      create_branch: if_missing
      commit: manual
      push: never
    workspace:
      mode: job
```

## Project Agent Profiles (`x-eve.agents`)

Defines harness profiles used by orchestration skills:

```yaml
x-eve:
  agents:
    version: 1
    availability:
      drop_unavailable: true
    profiles:
      primary-reviewer:
        - harness: mclaude
          model: opus-4.5
          reasoning_effort: high
        - harness: codex
          model: gpt-5.2-codex
          reasoning_effort: x-high
```

## AgentPacks (`x-eve.packs` + `x-eve.install_agents`)

AgentPacks import agent/team/chat config and skills from pack repos. Packs are
resolved by `eve agents sync` and locked in `.eve/packs.lock.yaml`.

```yaml
x-eve:
  install_agents: [claude-code, codex, gemini-cli]  # defaults to [claude-code]
  packs:
    - source: ./skillpacks/my-pack
    - source: incept5/eve-skillpacks
      ref: 0123456789abcdef0123456789abcdef01234567
    - source: ./skillpacks/claude-only
      install_agents: [claude-code]
```

Notes:
- Remote pack sources require a 40-char git SHA `ref`.
- Packs can be full AgentPacks (`eve/pack.yaml`) or skills-only packs.
- Local packs use relative paths (resolved from repo root).

### Pack Lock File

`.eve/packs.lock.yaml` tracks resolved state:

```yaml
resolved_at: "2026-02-09T..."
project_slug: myproject
packs:
  - id: pack-id
    source: incept5/eve-skillpacks
    ref: 0123456789abcdef0123456789abcdef01234567
    pack_version: 1
effective:
  agents_count: 5
  teams_count: 2
  routes_count: 3
  profiles_count: 4
```

### Pack Overlay Customization

Local YAML overlays pack defaults using deep merge + `_remove`:

```yaml
# In local agents.yaml
version: 1
agents:
  pack-agent:
    harness_profile: my-override       # override pack default
  unwanted-agent:
    _remove: true                       # remove from pack
```

### Pack CLI

```bash
eve packs status [--repo-dir <path>]           # Show lockfile + drift
eve packs resolve [--dry-run] [--repo-dir <path>]  # Preview resolution
```

## Secrets Requirements + Interpolation

Declare required secrets:

```yaml
x-eve:
  requires:
    secrets: [GITHUB_TOKEN, REGISTRY_TOKEN]
```

Interpolate secrets in env vars:

```yaml
environment:
  DATABASE_URL: postgres://user:${secret.DB_PASSWORD}@db:5432/app
```

Also supported (runtime interpolation): `${ENV_NAME}`, `${PROJECT_ID}`, `${ORG_ID}`, `${ORG_SLUG}`, `${COMPONENT_NAME}`.

## Ingress Defaults

If a service exposes ports and the cluster domain is configured, Eve creates ingress by default.
Set `x-eve.ingress.public: false` to disable.

URL pattern: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}`
