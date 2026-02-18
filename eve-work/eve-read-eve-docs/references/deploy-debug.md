# Deployment + Debugging (Current)

## Use When
- You need to troubleshoot deploy, ingress, namespace, or runtime worker issues.
- You need environment-specific diagnostics or service status during incident response.
- You need K8s architecture behavior for local or staging deployments.

## Load Next
- `references/cli.md` for command-based diagnostics.
- `references/pipelines-workflows.md` if the issue is pipeline-triggered.
- `references/builds-releases.md` for build/release failure context.

## Ask If Missing
- Confirm runtime mode (`k8s` vs `docker`) and `EVE_API_URL`.
- Confirm environment name, namespace, and whether this is staging or local.
- Confirm which command already ran and what exact output/error was returned.

## Default Environment

Default to **staging** for user guidance. Use local/docker only when explicitly asked for local development.

## Runtime Modes

| Mode | `EVE_RUNTIME` | Purpose | Runner Execution |
|------|---------------|---------|------------------|
| Kubernetes | `k8s` | Integration, staging, production | Ephemeral pods |
| Docker Compose | `docker` (default) | Local dev iteration | Local process |

Agent runtime hot path is configured separately with `EVE_AGENT_RUNTIME_EXECUTION_MODE`:
- `inline` (default): execute directly in warm runtime pods
- `runner`: fallback to per-attempt runner pod execution

## K8s Architecture

- **API, Orchestrator, Worker**: Deployments in the `eve` namespace (worker is not per-env).
- **Postgres**: StatefulSet with 5Gi PVC.
- **Runner pods**: Ephemeral pods spawned per job attempt for isolated execution.
- **Ingress**: Access via `http://api.eve.lvh.me` -- no port-forwarding needed.

When web auth is enabled, the stack also runs:
- **supabase-auth (GoTrue)** at `auth.<domain>`
- **sso** at `sso.<domain>`
- **mailpit** at `mail.<domain>` (local only)
- **auth bootstrap job** for GoTrue DB role provisioning

```bash
./bin/eh k8s start     # Start k3d cluster + apply manifests
./bin/eh k8s deploy    # Build images + deploy stack
./bin/eh k8s status    # Check status
```

### Runner Pod Reaper

The worker runs a periodic reaper that cleans up orphaned runner pods and PVCs after job completion. Prevents resource leaks if the worker restarts mid-poll. Settings: `EVE_RUNNER_REAPER_ENABLED`, `EVE_RUNNER_REAPER_INTERVAL_MS`, `EVE_RUNNER_REAPER_GRACE_SECONDS`.

### Secrets Provisioning

1. Define secrets in `system-secrets.env.local` (e.g., `GITHUB_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN`).
2. Run `./bin/eh k8s secrets` to sync `system-secrets.env.local` plus auth-derived keys into the K8s secret `eve-app`.
3. Restart deployments that consume `eve-app` (`eve-api`, `eve-orchestrator`, `eve-worker`) so updated env values are loaded.
4. API reads system secrets as baseline for all environments.

### Ingress Routing

```
Pattern:   {service}.{orgSlug}-{projectSlug}-{env}.{domain}
Example:   api.myorg-myproj-staging.eh1.incept5.dev
Namespace: eve-{orgSlug}-{projectSlug}-{env}
```

Domain resolution: 1) manifest `x-eve.ingress.domain`, 2) `EVE_DEFAULT_DOMAIN`, 3) no ingress if neither set. Local dev uses `lvh.me` (resolves to 127.0.0.1). Production: set `EVE_DEFAULT_DOMAIN=apps.yourdomain.com`.

### Ingress TLS

Use cert-manager for automatic TLS on app ingresses. Set `EVE_DEFAULT_TLS_CLUSTER_ISSUER` (e.g., `letsencrypt-prod`) to enable per-host certs via cert-manager annotations. Optionally set `EVE_DEFAULT_TLS_SECRET` for a wildcard cert or `EVE_DEFAULT_INGRESS_CLASS` for a specific ingress controller.

## Worker Image Registry

Pre-built images on public ECR eliminate local builds and ensure consistent toolchains.

| Image | Public ECR Path | Contents |
|-------|-----------|----------|
| base | `public.ecr.aws/w7c4v0w3/eve-horizon/worker-base:<ver>` | Node.js, worker harness, base utilities |
| python | `public.ecr.aws/w7c4v0w3/eve-horizon/worker-python:<ver>-py3.11` | Python 3.11, pip, uv |
| rust | `public.ecr.aws/w7c4v0w3/eve-horizon/worker-rust:<ver>-rust1.75` | Rust 1.75, cargo |
| java | `public.ecr.aws/w7c4v0w3/eve-horizon/worker-java:<ver>-jdk21` | OpenJDK 21 |
| kotlin | `public.ecr.aws/w7c4v0w3/eve-horizon/worker-kotlin:<ver>-kotlin2.0-jdk21` | Kotlin 2.0 + JDK 21 |
| full | `public.ecr.aws/w7c4v0w3/eve-horizon/worker-full:<ver>` | All toolchains (default) |

### Versioning + Pinning

- **Version tags**: `0.1.0`, `0.1.0-py3.11` (from git tag `worker-images/vX.Y.Z`).
- **SHA tags**: `sha-a1b2c3d` (every build, for commit-level pinning).
- **Multi-arch**: `linux/amd64` + `linux/arm64`.

Configure via `EVE_RUNNER_IMAGE` on the worker deployment. Pin to semantic versions in production. Use SHA tags or `latest` for dev/CI only.

## Docker Compose Runtime (Dev Only)

Optimized for fast local iteration. **Security note:** exposes services on localhost with simple defaults -- never run in shared or internet-exposed environments.

```bash
./bin/eh docker auth    # Extract auth credentials from host
./bin/eh start docker   # Start the stack
```

| Aspect | Docker Compose | K8s (k3d) |
|--------|----------------|-----------|
| Startup | ~10s | ~60s |
| Prod parity | Moderate | High |
| Runner pods | No (local process) | Yes (ephemeral) |

## First Deploy Quickstart

The fastest path from zero to a running deployment. Create a minimal `.eve/manifest.yaml`:

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

Then deploy with two commands:

```bash
eve project sync --dir .
eve env deploy sandbox --ref main
```

Key points:
- **`registry: "eve"`** uses the Eve-native container registry -- no external registry setup needed.
- **`image` is optional** when `build` and `registry` are configured -- the platform derives image names from service keys.
- **`eve env deploy` auto-creates** the `sandbox` environment because it is defined in `manifest.environments`. No separate `eve env create` step is required.
- The pipeline builds, releases, and deploys in sequence. Access the app at `http://app.{orgSlug}-{projectSlug}-sandbox.{domain}`.

## Deploying Environments

```bash
eve env deploy staging --ref main --repo-dir ./my-app
eve env deploy staging --ref <40-char-sha>
```

If `environments.<env>.pipeline` is set, `eve env deploy` triggers that pipeline. Use `--direct` to bypass. `--ref` must be a 40-char SHA or a ref resolved against `--repo-dir`/cwd. When `--repo-dir` is provided and the directory contains `.eve/manifest.yaml`, the CLI automatically syncs the manifest before deploying (see below).

### Manifest Auto-Sync on Deploy

When `--repo-dir` points to a repository containing `.eve/manifest.yaml`, the CLI automatically syncs the manifest to the API before deploying. This eliminates the separate `eve project sync` step:

1. CLI reads `.eve/manifest.yaml` from the repo directory.
2. POSTs the manifest YAML with the current git SHA and branch to sync.
3. Uses the returned manifest hash for the deploy request.

If no `.eve/manifest.yaml` exists in the repo directory, the CLI falls back to fetching the latest manifest hash from the server (previous behavior).

## Deploy Polling

### Starting a Deploy

```
POST /projects/{projectId}/envs/{envName}/deploy
```

Body requires `release_tag` **or** both `git_sha` + `manifest_hash`. Optional: `image_digests`, `image_tag`, `direct` (bypass pipeline), `inputs`.

### Response Discrimination

Check for `pipeline_run` in the response:
- **Present**: pipeline deploy -- poll the pipeline run, then health check.
- **Absent**: direct deploy -- skip to health check immediately.

### Pipeline Polling

```
GET /projects/{projectId}/pipelines/{pipelineName}/runs/{runId}
```

Terminal statuses: `succeeded` (proceed to health), `failed` (read errors), `cancelled` (aborted). Non-terminal: `pending`, `running`, `awaiting_approval`. Inspect individual steps on failure for `error_message`, `exit_code`, and `logs_ref`.

### Health Check

```
GET /projects/{projectId}/envs/{envName}/health
```

| Status | Meaning |
|--------|---------|
| `ready` | All pods healthy, no in-flight pipeline |
| `deploying` | Pods rolling out or pipeline active |
| `degraded` | Some pods unhealthy |
| `unknown` | K8s unavailable |

The health endpoint is **pipeline-aware**: reports `deploying` while a pipeline run is pending/running, preventing false `ready` from stale pods.

Deploy is complete when all three hold: `ready === true`, `active_pipeline_run === null`, `status === "ready"`.

### Polling Pseudocode

```
response = POST /projects/{id}/envs/{env}/deploy { ... }

if response.pipeline_run:
    run_id   = response.pipeline_run.run.id
    pipeline = response.pipeline_run.run.pipeline_name
    loop every 3-5s, timeout 300s:
        detail = GET /projects/{id}/pipelines/{pipeline}/runs/{run_id}
        if detail.run.status == "succeeded": break
        if detail.run.status in ("failed","cancelled"): FAIL
    loop every 3-5s, timeout 120s:
        health = GET /projects/{id}/envs/{env}/health
        if health.ready and not health.active_pipeline_run: SUCCESS
else:
    loop every 3-5s, timeout 120s:
        health = GET /projects/{id}/envs/{env}/health
        if health.ready: SUCCESS
```

## Environment Variable Interpolation

Manifest environment values support these interpolation variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `${ENV_NAME}` | Environment name | `staging` |
| `${PROJECT_ID}` | Project ID | `proj_01abc...` |
| `${ORG_ID}` | Organization ID | `org_01xyz...` |
| `${ORG_SLUG}` | Organization slug | `acme` |
| `${COMPONENT_NAME}` | Current service name | `api`, `web` |
| `${secret.KEY}` | Secret value | `${secret.DB_PASSWORD}` |
| `${managed.<service>.<field>}` | Managed DB value (when provisioned) | `${managed.db.url}` |

## Platform Env Vars (Injected into Deployed Apps)

| Variable | Purpose |
|----------|---------|
| `EVE_API_URL` | Internal cluster URL for server-to-server calls |
| `EVE_PUBLIC_API_URL` | Public ingress URL for browser-facing apps |
| `EVE_PROJECT_ID` | Current project ID |
| `EVE_ORG_ID` | Current org ID |
| `EVE_ENV_NAME` | Current environment name |

Use `EVE_API_URL` for backend calls from containers. Use `EVE_PUBLIC_API_URL` for browser/client-side code.

## Runtime Environment Variables

| Variable | Component | Purpose |
|----------|-----------|---------|
| `ORCH_LOOP_INTERVAL_MS` | Orchestrator | Main claim/dispatch loop cadence |
| `ORCH_CONCURRENCY` | Orchestrator | Base concurrency |
| `ORCH_CONCURRENCY_MIN` / `_MAX` | Orchestrator | Tuner bounds |
| `ORCH_TUNER_ENABLED` | Orchestrator | Enable adaptive tuning |
| `ORCH_TUNER_INTERVAL_MS` | Orchestrator | Tuning check interval |
| `ORCH_TUNER_CPU_THRESHOLD` | Orchestrator | CPU threshold for scaling |
| `ORCH_TUNER_MEMORY_THRESHOLD` | Orchestrator | Memory threshold for scaling |
| `EVE_WORKER_POLL_INTERVAL_MS` | Orchestrator | Poll cadence for worker completion events |
| `EVE_AGENT_RUNTIME_POLL_INTERVAL_MS` | Orchestrator | Poll cadence for agent-runtime completion events |
| `EVE_RUNNER_IMAGE` | Worker | Container image for runner pods |
| `EVE_RUNNER_REAPER_ENABLED` | Worker | Enable pod reaper |
| `EVE_RUNNER_REAPER_INTERVAL_MS` | Worker | Reaper sweep interval |
| `EVE_RUNNER_REAPER_GRACE_SECONDS` | Worker | Grace before cleanup |
| `EVE_WORKSPACE_MAX_GB` | Worker | Total workspace budget per instance |
| `EVE_WORKSPACE_MIN_FREE_GB` | Worker | Hard floor; refuse new claims below |
| `EVE_WORKSPACE_TTL_HOURS` | Worker | Idle TTL for job worktrees |
| `EVE_SESSION_TTL_HOURS` | Worker | Idle TTL for session workspaces |
| `EVE_MIRROR_MAX_GB` | Worker | Cap for bare mirrors |

## Workspace Janitor (Disk Safety)

Policies for production disk management:
- LRU eviction of worktrees when over budget.
- TTL cleanup for idle job/session worktrees.
- Mirror maintenance: `git fetch --prune` + periodic `git gc --prune=now`.
- Emit system events on low disk; refuse new attempts below minimum free space.

K8s: per-attempt PVCs are deleted after completion. Session-scoped PVCs require TTL cleanup and storage quotas.

## CLI-First Debugging Ladder

1. **CLI first** -- `eve system health`, `eve job diagnose <id>`, `eve job follow <id>`.
2. **Environment status** -- `./bin/eh status` to confirm URLs and running services.
3. **kubectl** only if CLI lacks data.

CLI only needs `EVE_API_URL`: local/docker = `http://localhost:4801`, K8s = `http://api.eve.lvh.me`.

## Debugging Workflows

### Job Won't Start

```bash
eve job show <id> --verbose    # Check phase
eve job dep list <id>          # Check blocked deps
eve job ready --project <id>   # Check ready queue
```

Common causes: phase is not `ready`, blocked by dependencies, orchestrator unhealthy.

### Job Failed

```bash
eve job diagnose <id>          # Status, timeline, attempts, errors, recommendations
eve job logs <id> --attempt N  # Detailed logs for specific attempt
```

`eve job diagnose` also shows LLM routing metadata for managed models (direct/bridge,
bridge ID, protocol pair, effective base URL).

### Job Stuck Active

Run `eve job diagnose <id>` and look for "running for Xs - may be stuck". Possible causes: harness hanging, worker crashed without marking attempt failed, network issues.

### System Issues

```bash
eve system health              # Quick health check
eve system logs api            # API pod logs
eve system logs orchestrator   # Orchestrator logs
eve system logs worker         # Worker logs
eve system logs postgres       # DB logs
```

If API is unhealthy: check API logs, verify database, confirm `EVE_API_URL`.

### Build Debugging

```bash
eve build list --project <id>       # Recent builds
eve build diagnose <build_id>       # Spec + runs + artifacts + logs
eve build logs <build_id>           # Raw build output
```

Common issues: registry auth (`registry: "eve"` for managed apps; `REGISTRY_USERNAME` + `REGISTRY_PASSWORD` only for BYO/custom registry), Dockerfile path (`build.context`), build backend (BuildKit on K8s, Buildx locally).

## Real-time Debugging

### Three-Terminal Approach

```bash
# Terminal 1: Poll status
watch -n 5 'eve job show <id> --verbose 2>&1 | head -30'

# Terminal 2: Stream harness logs
eve job follow <id>

# Terminal 3 (K8s): Watch runner pod
eve job runner-logs <id>
```

| Command | What It Shows |
|---------|---------------|
| `eve job watch <id>` | Combined status + logs streaming |
| `eve job follow <id>` | Harness JSONL logs (SSE) -- harness output only |
| `eve job runner-logs <id>` | K8s runner pod stdout/stderr |
| `eve job wait <id> --verbose` | Status changes while waiting |

Startup errors (clone, workspace, auth) appear in orchestrator/worker/runner logs, not in `follow`.

### Auth / Secrets Failures

```bash
eve secrets show GITHUB_TOKEN --project <proj_id>
eve secrets list --project <proj_id>
```

Check orchestrator/worker logs for `[resolveSecrets]` warnings. Verify `EVE_INTERNAL_API_KEY` and `EVE_SECRETS_MASTER_KEY` are set.

## Common Error Messages

| Error | Meaning | Fix |
|-------|---------|-----|
| "OAuth token has expired" | Claude auth stale | `./bin/eh auth extract --save` then redeploy |
| "No protocol bridge..." | Harness/provider protocol mismatch has no bridge route | Align managed model harness/provider or add a matching bridge |
| "Bridge ... missing base URL/API key" | Bridge runtime env is not configured | Set `EVE_BRIDGE_LITELLM_ANTHROPIC_OPENAI_URL` and `EVE_BRIDGE_LITELLM_ANTHROPIC_OPENAI_KEY` |
| "git clone failed" | Repo inaccessible | Check GITHUB_TOKEN secret |
| "Service X ready check failed" | Service provisioning issue | Check manifest services, container logs |
| "Orchestrator restarted while attempt was running" | Job orphaned on restart | Auto-retries via recovery |

## Environment-Specific Debugging

**Local dev:** `./bin/eh start local` -- logs at `/tmp/eve-{api,orchestrator,worker}.log`.

**Docker Compose:** `./bin/eh start docker` -- use `docker logs eve-api -f`, `docker logs eve-orchestrator -f`, `docker logs eve-worker -f`.

**Kubernetes:** `./bin/eh k8s start` + `export EVE_API_URL=http://api.eve.lvh.me`. Use `kubectl -n eve get pods | grep runner` and `kubectl -n eve logs -f eve-runner-<attempt-id>` only as last resort.
