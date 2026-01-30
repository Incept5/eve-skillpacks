# Manifest v2 (Current)

The manifest (`.eve/manifest.yaml`) is the single source of truth for builds, deploys, pipelines, and workflows.
Schema is Compose-like with Eve extensions under `x-eve`.

## Top-Level Fields

```yaml
schema: eve/compose/v1          # optional
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

- `role`: `component` (default), `worker`, or `job`
- `ingress`: `{ public: true|false, port: number }`
- `api_spec` / `api_specs`: register API specs (OpenAPI)
- `external`: true for external deps (not deployed)
- `connection_url`: connection string for external services
- `worker_type`: worker pool type for this service

Notes:
- `x-eve.role: job` makes a service runnable as a one-off job (migrations, seeds).
- `spec_url` can be relative (resolved against service URL) or absolute.
- `spec_path` is supported only for local `file://` repos.

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
