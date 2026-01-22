---
name: eve-manifest-authoring
description: Author and maintain Eve manifest files (.eve/manifest.yaml) for components, environments, pipelines, workflows, and secret interpolation. Use when changing deployment shape or runtime configuration in an Eve-compatible repo.
---

# Eve Manifest Authoring

Keep the manifest as the single source of truth for build and deploy behavior.

## Minimal skeleton

```yaml
name: my-project

registry:
  host: ghcr.io
  namespace: myorg
  auth:
    username_secret: GHCR_USERNAME
    token_secret: GHCR_TOKEN

components:
  api:
    image: ghcr.io/myorg/my-project-api
    build:
      context: ./apps/api
    port: 3000

environments:
  test:
    pipeline: deploy-test

pipelines:
  deploy-test:
    steps:
      - name: build
        action:
          type: build
      - name: deploy
        depends_on: [build]
        action:
          type: deploy
```

## Components

- Provide `image` and optionally `build` (context and dockerfile).
- Use `port`, `env`, `healthcheck`, `depends_on`, and `migrations` as needed.
- Use `external: true` and `connection_url` for externally hosted services.

## Environments, pipelines, workflows

- Link each environment to a pipeline.
- Pipeline steps can be `action`, `script`, or `agent`.
- Workflows live under `workflows` and are invoked via CLI; trigger blocks exist but auto triggers are planned.

## Interpolation and secrets

- Env interpolation: `${ENV_NAME}`, `${PROJECT_ID}`, `${ORG_ID}`, `${COMPONENT_NAME}`.
- Secret interpolation: `${secret.KEY}` pulls from Eve secrets or `.eve/secrets.yaml`.
- Use `.eve/secrets.yaml` for local overrides; set real secrets via the API for production.

## Recursive skill distillation

- Add new manifest patterns and pitfalls as they emerge.
- Split deep details into a `references/` file if this skill grows.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
