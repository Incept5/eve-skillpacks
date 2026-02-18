# Manifest (Current)

## Use When
- You need to author, validate, or review `.eve/manifest.yaml`.
- You need to configure services, environments, pipelines, or harness defaults.
- You need to prepare manifest changes for deployable, reproducible builds.

## Load Next
- `references/pipelines-workflows.md` for pipeline/job wiring in manifests.
- `references/secrets-auth.md` for secret declaration and resolution order.
- `references/overview.md` for core platform concepts before editing complex files.

## Ask If Missing
- Confirm target manifest path and environment names.
- Confirm whether managed DBs, external services, or custom ingress are required.
- Confirm any required repository path, branch, or org/project identifiers.

The manifest (`.eve/manifest.yaml`) is the single source of truth for builds, deploys, pipelines, and workflows.
Schema is Compose-like with Eve extensions under `x-eve`.

## Minimal Example

The simplest possible deployable project. Uses the Eve-native registry so `image` fields are auto-derived from service keys:

```yaml
schema: eve/compose/v2
project: my-app

registry: "eve"

services:
  app:
    build:
      context: .
    ports: ["3000"]
    x-eve:
      ingress:
        public: true

environments:
  sandbox:
    pipeline: deploy

pipelines:
  deploy:
    steps:
      - name: build
        action: { type: build }
      - name: release
        depends_on: [build]
        action: { type: release }
      - name: deploy
        depends_on: [release]
        action: { type: deploy, env_name: sandbox }
```

Deploy with two commands:

```bash
eve project sync --dir .
eve env deploy sandbox --ref main
```

## Full Example

A more complete manifest showing registry auto-derivation, a database with healthcheck, an API with ingress, a migration job, environment overrides, and a simple pipeline:

```yaml
schema: eve/compose/v2
project: my-project

registry: "eve"

services:
  db:
    image: postgres:16
    ports: [5432]
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${secret.DB_PASSWORD}
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "app"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build:
      context: ./apps/api
    ports: [3000]
    environment:
      DATABASE_URL: postgres://app:${secret.DB_PASSWORD}@db:5432/app
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

  migrate:
    image: flyway/flyway:10
    command: -url=jdbc:postgresql://db:5432/app -user=app -password=${secret.DB_PASSWORD} -locations=filesystem:/migrations migrate
    volumes:
      - ./db/migrations:/migrations:ro
    depends_on:
      db:
        condition: service_healthy
    x-eve:
      role: job

environments:
  sandbox:
    pipeline: deploy-sandbox
    overrides:
      services:
        api:
          environment:
            NODE_ENV: test

pipelines:
  deploy-sandbox:
    steps:
      - name: migrate
        action: { type: job, service: migrate }
      - name: deploy
        depends_on: [migrate]
        action: { type: deploy }
```

## Top-Level Fields

```yaml
schema: eve/compose/v2          # optional schema identifier
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
registry: "eve"   # Default Eve-managed registry
```

Use `registry: "eve"` unless your app must publish to a BYO registry.

For a private custom registry, switch to full object form:

```yaml
registry:
  host: public.ecr.aws/w7c4v0w3
  namespace: myorg
  auth:
    username_secret: REGISTRY_USERNAME
    token_secret: REGISTRY_PASSWORD
```

The deployer uses these secrets to create Kubernetes `imagePullSecrets` for private BYO registries. See container registry reference for setup details.

String modes:
```yaml
registry: "eve"   # Use Eve-managed registry (default)
registry: "none"  # Disable registry handling
registry:           # BYO registry (full object; see above)
```

## Services (Compose-Style)

```yaml
services:
  api:
    build:
      context: ./apps/api
    # image omitted (auto-derived as "api" when build is present)
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

**Image auto-derivation**: When a service has `build` config and a `registry` is configured, the `image` field is optional. With Eve-managed default (`registry: "eve"`), platform derives the image name from the service key (for example, service `app` becomes `image: app`) and prefixes it at build time with the managed registry host.

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
- If a service exposes ports and the cluster domain is configured, Eve creates ingress by default. Set `x-eve.ingress.public: false` to disable.

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

## Platform Environment Variables

Eve automatically injects these variables into all deployed services:

| Variable | Description |
|----------|-------------|
| `EVE_API_URL` | Internal cluster URL for server-to-server calls (e.g., `http://eve-api:4701`) |
| `EVE_PUBLIC_API_URL` | Public ingress URL for browser-facing apps (e.g., `https://api.eh1.incept5.dev`) |
| `EVE_PROJECT_ID` | The project ID (e.g., `proj_01abc123...`) |
| `EVE_ORG_ID` | The organization ID (e.g., `org_01xyz789...`) |
| `EVE_ENV_NAME` | The environment name (e.g., `staging`, `production`) |

Job runners also receive `EVE_ENV_NAMESPACE`, but service containers do not.
Services can override these values by defining them explicitly in their `environment` section.

**Which API URL to use:**

- Use `EVE_API_URL` for backend/server-side calls from your container to the Eve API (internal cluster networking).
- Use `EVE_PUBLIC_API_URL` for browser/client-side calls or any code running outside the cluster.

```javascript
// Server-side: call Eve API from your backend
const eveApiUrl = process.env.EVE_API_URL;

// Client-side: expose to browser for frontend API calls
const publicApiUrl = process.env.EVE_PUBLIC_API_URL;
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

### Environment Fields

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `persistent` (default) or `temporary` |
| `kind` | string | `standard` (default) or `preview` (PR envs) |
| `pipeline` | string | Pipeline name to trigger on deploy |
| `pipeline_inputs` | object | Inputs passed to pipeline (CLI `--inputs` wins on conflict) |
| `approval` | string | `required` to gate deploys |
| `overrides` | object | Compose-style service overrides |
| `workers` | array | Worker pool configuration |
| `labels` | object | Metadata (PR info for preview envs) |

### Environment Pipeline Behavior

When `pipeline` is configured for an environment, `eve env deploy <env> --ref <sha>` triggers a pipeline run instead of performing a direct deployment. This enables:

- Consistent build/test/deploy workflows across environments
- Promotion patterns where staging/production reuse releases from test
- Environment-specific pipeline inputs and approval gates

To bypass the pipeline and perform a direct deployment, use `--direct`:

```bash
eve env deploy staging --ref 0123456789abcdef0123456789abcdef01234567 --direct
```

### Promotion Example

Define environments that share a pipeline but vary in inputs and approval gates:

```yaml
environments:
  test:
    pipeline: deploy-test
  staging:
    pipeline: deploy
    pipeline_inputs:
      smoke_test: true
  production:
    pipeline: deploy
    approval: required
```

Deploy flow:

```bash
# Build + test + release in test
eve env deploy test --ref 0123456789abcdef0123456789abcdef01234567

# Promote to staging (reuse release, no rebuild)
eve release resolve v1.2.3  # Get release_id from test
eve env deploy staging --ref 0123456789abcdef0123456789abcdef01234567 --inputs '{"release_id":"rel_xxx"}'

# Promote to production (approval required)
eve env deploy production --ref 0123456789abcdef0123456789abcdef01234567 --inputs '{"release_id":"rel_xxx"}'
```

This pattern enables build-once, deploy-many promotion workflows without rebuilding images.

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

## Secret Requirements and Validation

Declare required secrets at the top level or per pipeline step:

```yaml
x-eve:
  requires:
    secrets: [GITHUB_TOKEN, REGISTRY_TOKEN]

pipelines:
  ci-cd-main:
    steps:
      - name: integration-tests
        script:
          run: "pnpm test"
        requires:
          secrets: [DATABASE_URL]
```

Validate secrets before syncing:

```bash
eve project sync --validate-secrets     # Warn on missing secrets
eve project sync --strict               # Fail on missing secrets
eve manifest validate                   # Schema + secret validation without syncing
```

Use `eve manifest validate` for pre-flight checks against a local manifest or the latest synced version. Required keys follow standard scope resolution rules.

### Secret Interpolation

Interpolate secrets in environment variables:

```yaml
environment:
  DATABASE_URL: postgres://user:${secret.DB_PASSWORD}@db:5432/app
```

Also supported (runtime interpolation): `${ENV_NAME}`, `${PROJECT_ID}`, `${ORG_ID}`, `${ORG_SLUG}`, `${COMPONENT_NAME}`, `${secret.KEY}`, `${managed.<service>.<field>}`.

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
      resource_class: job.c1
      max_cost:
        currency: usd
        amount: 5
      max_tokens: 200000
    git:
      ref_policy: auto
      branch: job/${job_id}
      create_branch: if_missing
      commit: manual
      push: never
    workspace:
      mode: job
```

`hints` can include budgeting and accounting fields such as `resource_class`,
`max_cost`, and `max_tokens`. These map to scheduling hints and per-attempt
budget enforcement.

## Project Agent Profiles (`x-eve.agents`)

Define harness profiles used by orchestration skills:

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

## Project Bootstrap

Bootstrap creates a project + environments in a single API call:

```bash
eve project bootstrap --name my-app --repo-url https://github.com/org/repo \
  --environments staging,production
```

API: `POST /projects/bootstrap` with body:
- `org_id`, `name`, `repo_url`, `branch` (required)
- `slug`, `description`, `template`, `packs`, `environments` (optional)

Idempotent — re-calling with the same name returns the existing project.

## Ingress Defaults

If a service exposes ports and the cluster domain is configured, Eve creates ingress by default.
Set `x-eve.ingress.public: false` to disable.

URL pattern: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}`
