---
name: eve-fullstack-app-design
description: Architect a full-stack application on Eve Horizon — manifest-driven services, managed databases, build pipelines, deployment strategies, secrets, and observability. Use when designing a new app, planning a migration, or evaluating your architecture.
triggers:
  - fullstack design
  - app architecture
  - design an app
  - architect app
  - fullstack app
  - app design
  - system design eve
---

# Full-Stack App Design on Eve Horizon

Architect applications where the manifest is the blueprint, the platform handles infrastructure, and every design decision is intentional.

## When to Use

Load this skill when:
- Designing a new application from scratch on Eve
- Migrating an existing app onto the platform
- Evaluating whether your current architecture uses Eve's capabilities well
- Planning service topology, database strategy, or deployment pipelines
- Deciding between managed and external services

This skill teaches *design thinking* for Eve's PaaS layer. For CLI usage and operational detail, load the corresponding eve-se skills (`eve-manifest-authoring`, `eve-deploy-debugging`, `eve-auth-and-secrets`, `eve-pipelines-workflows`).

## The Manifest as Blueprint

The manifest (`.eve/manifest.yaml`) is the single source of truth for your application's shape. Treat it as an architectural document, not just configuration.

### What the Manifest Declares

| Concern | Manifest Section | Design Decision |
|---------|-----------------|-----------------|
| Service topology | `services` | What processes run, how they connect |
| Infrastructure | `services[].x-eve` | Managed DB, ingress, roles |
| Build strategy | `services[].build` + `registry` | What gets built, where images live |
| Release pipeline | `pipelines` | How code flows from commit to production |
| Environment shape | `environments` | Which environments exist, what pipelines they use |
| Agent configuration | `x-eve.agents`, `x-eve.chat` | Agent profiles, team dispatch, chat routing |
| Runtime defaults | `x-eve.defaults` | Harness, workspace, git policies |

**Design principle**: If an agent or operator can't understand your app's shape by reading the manifest, the manifest is incomplete.

## Service Topology

### Choose Your Services

Most Eve apps follow one of these patterns:

**API + Database** (simplest):
```
services:
  api:        # HTTP service with ingress
  db:         # managed Postgres
```

**API + Worker + Database**:
```
services:
  api:        # HTTP service (user-facing)
  worker:     # Background processor (jobs, queues)
  db:         # managed Postgres
```

**Multi-Service**:
```
services:
  web:        # Frontend/SSR
  api:        # Backend API
  worker:     # Background jobs
  db:         # managed Postgres
  redis:      # external cache (x-eve.external: true)
```

### Service Design Rules

1. **One concern per service.** Separate HTTP serving from background processing. An API service should not also run scheduled jobs.
2. **Use managed DB for Postgres.** Declare `x-eve.role: managed_db` and let the platform provision, connect, and inject credentials. No manual connection strings.
3. **Mark external services explicitly.** Use `x-eve.external: true` with `x-eve.connection_url` for services hosted outside Eve (Redis, third-party APIs).
4. **Use `x-eve.role: job` for one-off tasks.** Migrations, seeds, and data backfills are job services, not persistent processes.
5. **Expose ingress intentionally.** Only services that need external HTTP access get `x-eve.ingress.public: true`. Internal services communicate via cluster networking.

### Platform-Injected Variables

Every deployed service receives `EVE_API_URL`, `EVE_PUBLIC_API_URL`, `EVE_PROJECT_ID`, `EVE_ORG_ID`, and `EVE_ENV_NAME`. Use `EVE_API_URL` for server-to-server calls. Use `EVE_PUBLIC_API_URL` for browser-facing code. Design your app to read these rather than hardcoding URLs.

## Database Design

### Provisioning

Declare a managed database in the manifest:

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

Reference the connection URL in other services: `${managed.db.url}`.

### Schema Strategy

1. **Migrations are first-class.** Use `eve db new` to create migration files. Use `eve db migrate` to apply them. Never modify production schemas by hand.
2. **Design for RLS from the start.** If agents or users will query the database directly, scaffold RLS helpers early: `eve db rls init --with-groups`. Retrofitting row-level security is painful.
3. **Inspect before changing.** Use `eve db schema` to examine current schema. Use `eve db sql --env <env>` for ad-hoc queries during development. Use `--direct-url` mode for local dev tools that need a raw connection string.
4. **Separate app data from agent data.** Use distinct schemas or naming conventions. App tables serve the product; agent tables serve memory and coordination (see `eve-agent-memory` for storage patterns).

### Access Patterns

| Who Queries | How | Auth |
|-------------|-----|------|
| App service | `${managed.db.url}` in service env | Connection string injected at deploy |
| Agent via CLI | `eve db sql --env <env>` | Job token scopes access |
| Agent via RLS | SQL with `app.current_user_id()` | Session context set by runtime |

## Build and Release Pipeline

### The Canonical Flow

Every production app should follow `build -> release -> deploy`:

```yaml
pipelines:
  deploy:
    steps:
      - name: build
        action:
          type: build        # Creates BuildSpec + BuildRun, produces image digests
      - name: release
        depends_on: [build]
        action:
          type: release      # Creates immutable release from build artifacts
      - name: deploy
        depends_on: [release]
        action:
          type: deploy       # Deploys release to target environment
```

**Why this matters**: The build step produces SHA256 image digests. The release step pins those exact digests. The deploy step uses the pinned release. You deploy exactly what you built — no tag drift, no "latest" surprises.

### Registry Decisions

| Option | When to Use |
|--------|-------------|
| `registry: "eve"` | Default. Internal registry with JWT auth. Simplest setup. |
| BYO registry (GHCR, ECR) | When you need images accessible outside Eve, or have existing CI. |
| `registry: "none"` | Public base images only. No custom builds. |

For GHCR, add OCI labels to Dockerfiles for automatic repository linking:
```dockerfile
LABEL org.opencontainers.image.source="https://github.com/YOUR_ORG/YOUR_REPO"
```

### Build Configuration

Every service with a custom image needs a `build` section:

```yaml
services:
  api:
    build:
      context: ./apps/api
      dockerfile: Dockerfile
    image: ghcr.io/org/my-api
```

Use multi-stage Dockerfiles. BuildKit handles them natively. Place the OCI label on the final stage.

## Deployment and Environments

### Environment Strategy

| Environment | Type | Purpose | Pipeline |
|-------------|------|---------|----------|
| `staging` | persistent | Integration testing, demos | `deploy` |
| `production` | persistent | Live traffic | `deploy` (with promotion) |
| `preview-*` | temporary | PR previews, feature branches | `deploy` (auto-cleanup) |

Link each environment to a pipeline in the manifest:

```yaml
environments:
  staging:
    pipeline: deploy
  production:
    pipeline: deploy
```

### Deployment Patterns

**Standard deploy**: `eve env deploy staging --ref main --repo-dir .` triggers the linked pipeline.

**Direct deploy** (bypass pipeline): `eve env deploy staging --ref <sha> --direct` for emergencies or simple setups.

**Promotion**: Build once in staging, then promote the same release artifacts to production. The build step's digests carry forward, guaranteeing identical images.

### Recovery

When a deploy fails:
1. **Diagnose**: `eve env diagnose <project> <env>` — shows health, recent deploys, service status.
2. **Logs**: `eve env logs <project> <env>` — container output.
3. **Rollback**: Redeploy the previous known-good release.
4. **Reset**: `eve env reset <project> <env>` — nuclear option, reprovisions from scratch.

Design your app to be rollback-safe: migrations should be forward-compatible, and services should handle schema version mismatches gracefully during rolling deploys.

## Secrets and Configuration

### Scoping Model

Secrets resolve with cascading precedence: **project > user > org > system**. A project-level `API_KEY` overrides an org-level `API_KEY`.

### Design Rules

1. **Set secrets per-project.** Use `eve secrets set KEY "value" --project proj_xxx`. Keep project secrets self-contained.
2. **Use interpolation in the manifest.** Reference `${secret.KEY}` in service environment blocks. The platform resolves at deploy time.
3. **Validate before deploying.** Run `eve manifest validate --validate-secrets` to catch missing secret references before they cause deploy failures.
4. **Use `.eve/dev-secrets.yaml` for local development.** Mirror the production secret keys with local values. This file is gitignored.
5. **Never store secrets in environment variables directly.** Always use `${secret.KEY}` interpolation. This ensures secrets flow through the platform's resolution and audit chain.

### Git Credentials

Agents need repository access. Set either `github_token` (HTTPS) or `ssh_key` (SSH) as project secrets. The worker injects these automatically during git operations.

## Observability and Debugging

### The Debugging Ladder

Escalate through these stages:

```
1. Status    → eve env show <project> <env>
2. Diagnose  → eve env diagnose <project> <env>
3. Logs      → eve env logs <project> <env>
4. Pipeline  → eve pipeline logs <pipeline> <run-id> --follow
5. Recover   → eve env deploy (rollback) or eve env reset
```

Start at the top. Each stage provides more detail and more cost. Most issues resolve at stages 1-2.

### Pipeline Observability

Monitor pipeline execution in real time:

```bash
eve pipeline logs <pipeline> <run-id> --follow         # stream all steps
eve pipeline logs <pipeline> <run-id> --follow --step build  # stream one step
```

Failed steps include failure hints and link to build diagnostics when applicable.

### Build Debugging

When builds fail:

```bash
eve build list --project <project_id>
eve build diagnose <build_id>
eve build logs <build_id>
```

Common causes: missing registry credentials, Dockerfile path mismatch, build context too large.

### Health Checks

Design services with health endpoints. Eve polls health to determine deployment readiness. A deploy is complete when `ready === true` and `active_pipeline_run === null`.

## Design Checklist

**Service Topology:**
- [ ] Each service has one responsibility
- [ ] Managed DB declared for Postgres needs
- [ ] External services marked with `x-eve.external: true`
- [ ] Only public-facing services have ingress enabled
- [ ] Platform-injected env vars used (not hardcoded URLs)

**Database:**
- [ ] Migrations managed via `eve db new` / `eve db migrate`
- [ ] RLS scaffolded if agents or users query directly
- [ ] App data separated from agent data by schema or convention

**Pipeline:**
- [ ] Canonical `build -> release -> deploy` pipeline defined
- [ ] Registry chosen and credentials set as secrets
- [ ] OCI labels on Dockerfiles (for GHCR)
- [ ] Image digests flow through release (no tag-based deploys)

**Environments:**
- [ ] Staging and production environments defined
- [ ] Each environment linked to a pipeline
- [ ] Promotion workflow defined (build once, deploy many)
- [ ] Recovery procedure known (diagnose -> rollback -> reset)

**Secrets:**
- [ ] All secrets set per-project via `eve secrets set`
- [ ] Manifest uses `${secret.KEY}` interpolation
- [ ] `eve manifest validate --validate-secrets` passes
- [ ] `.eve/dev-secrets.yaml` exists for local development
- [ ] Git credentials (`github_token` or `ssh_key`) configured

**Observability:**
- [ ] Services expose health endpoints
- [ ] The debugging ladder is understood (status -> diagnose -> logs -> recover)
- [ ] Pipeline logs are accessible via `eve pipeline logs --follow`

## Cross-References

- **Manifest syntax and options**: `eve-manifest-authoring`
- **Deploy commands and error resolution**: `eve-deploy-debugging`
- **Secret management and access groups**: `eve-auth-and-secrets`
- **Pipeline and workflow definitions**: `eve-pipelines-workflows`
- **Local development workflow**: `eve-local-dev-loop`
- **Layering agentic capabilities onto this foundation**: `eve-agentic-app-design`
