---
name: eve-manifest-authoring
description: Author and maintain Eve manifest files (.eve/manifest.yaml) for services, environments, pipelines, workflows, and secret interpolation. Use when changing deployment shape or runtime configuration in an Eve-compatible repo.
---

# Eve Manifest Authoring

Keep the manifest as the single source of truth for build and deploy behavior.

## Minimal skeleton (v2)

```yaml
schema: eve/compose/v1
project: my-project

registry:
  host: ghcr.io
  namespace: myorg
  auth:
    username_secret: GHCR_USERNAME
    token_secret: GHCR_TOKEN

## OCI Image Labels (GHCR Auto-Linking)

GHCR requires packages to be linked to a repository for proper permission inheritance. Add these labels to your Dockerfiles to enable automatic linking:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/YOUR_ORG/YOUR_REPO"
LABEL org.opencontainers.image.description="Service description"
```

**Why this matters**: Without this label, GHCR creates "orphaned" packages that only org admins can push to. The Eve builder injects this label automatically at build time, but including it in your Dockerfile is recommended as defense-in-depth.

For multi-stage Dockerfiles, add the labels to the **final** stage (the production image).

services:
  api:
    build:
      context: ./apps/api           # Build context directory
      dockerfile: Dockerfile        # Optional, defaults to context/Dockerfile
    image: ghcr.io/myorg/my-api     # Target image (no tag needed - managed by Eve)
    ports: [3000]
    environment:
      NODE_ENV: production
    x-eve:
      ingress:
        public: true
        port: 3000

environments:
  staging:
    pipeline: deploy
    pipeline_inputs:
      some_key: default_value

pipelines:
  deploy:
    steps:
      - name: build
        action:
          type: build               # Builds all services with build: config
      - name: release
        depends_on: [build]
        action:
          type: release
      - name: deploy
        depends_on: [release]
        action:
          type: deploy
```

## Legacy manifests

If the repo still uses `components:` from older manifests, migrate to `services:`
and add `schema: eve/compose/v1`. Keep ports and env keys the same.

## Services

- Provide `image` and optionally `build` (context and dockerfile).
- Use `ports`, `environment`, `healthcheck`, `depends_on` as needed.
- Use `x-eve.external: true` and `x-eve.connection_url` for externally hosted services.
- Use `x-eve.role: job` for one-off services (migrations, seeds).

### Build configuration

Services with Docker images should define their build configuration:

```yaml
services:
  api:
    build:
      context: ./apps/api           # Build context directory
      dockerfile: Dockerfile        # Optional, defaults to context/Dockerfile
    image: ghcr.io/org/my-api       # Target image (no tag needed - managed by Eve)
    ports: [3000]
```

Note: Every deploy pipeline should include a `build` step before `release`. The build step creates tracked BuildSpec/BuildRun records and produces image digests that releases use for deterministic deployments.

## Local dev alignment

- Keep service names and ports aligned with Docker Compose.
- Prefer `${secret.KEY}` and use `.eve/secrets.yaml` for local values.

## Environments, pipelines, workflows

- Link each environment to a pipeline via `environments.<env>.pipeline`.
- When `pipeline` is set, `eve env deploy <env>` triggers that pipeline instead of direct deploy.
- Use `environments.<env>.pipeline_inputs` to provide default inputs for pipeline runs.
- Override inputs at runtime with `eve env deploy <env> --ref <sha> --inputs '{"key":"value"}'`.
- Use `--direct` flag to bypass pipeline and do direct deploy: `eve env deploy <env> --ref <sha> --direct`.
- Pipeline steps can be `action`, `script`, or `agent`.
- Use `action.type: create-pr` for PR automation when configured.
- Workflows live under `workflows` and are invoked via CLI; `db_access` is honored.

## Interpolation and secrets

- Env interpolation: `${ENV_NAME}`, `${PROJECT_ID}`, `${ORG_ID}`, `${ORG_SLUG}`, `${COMPONENT_NAME}`.
- Secret interpolation: `${secret.KEY}` pulls from Eve secrets or `.eve/secrets.yaml`.
- Use `.eve/secrets.yaml` for local overrides; set real secrets via the API for production.

## Eve extensions

- Top-level defaults via `x-eve.defaults` (env, harness, harness_profile, harness_options, hints, git, workspace).
- Top-level agent policy via `x-eve.agents` (profiles, councils, availability rules).
- Service extensions under `x-eve` (ingress, role, api specs, worker pools).
- API specs: `x-eve.api_spec` or `x-eve.api_specs` (spec URL relative to service by default).

## Recursive skill distillation

- Add new manifest patterns and pitfalls as they emerge.
- Split deep details into a `references/` file if this skill grows.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
