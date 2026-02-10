# Builds + Releases (Current)

## Build Model

Builds are first-class primitives that track container image construction. The model is three-tier:

```
BuildSpec (immutable input) → BuildRun (execution) → BuildArtifact (output)
```

### BuildSpec

Created once per unique build configuration. Immutable.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | `bld_xxx` |
| `project_id` | string | Owning project |
| `git_sha` | string | 40-char commit SHA |
| `manifest_hash` | string | Hash of manifest at build time |
| `services` | string[]? | Service names to build (null = all) |
| `inputs` | object? | Additional build inputs |
| `registry` | object? | Registry configuration override |
| `cache` | object? | Cache configuration |
| `created_by` | string? | User/system that created the build |

### BuildRun

Execution instance of a build. Multiple runs can exist per spec (retries).

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Run identifier |
| `build_id` | string | Parent BuildSpec |
| `status` | enum | `pending` → `building` → `completed` / `failed` / `cancelled` |
| `backend` | string | `buildkit` (K8s), `buildx` (local), `kaniko` (fallback) |
| `runner_ref` | string? | Pod/runner name |
| `logs_ref` | string? | Log storage reference |
| `error_message` | string? | Failure reason |

### BuildArtifact

Output images from a successful build.

| Field | Type | Description |
|-------|------|-------------|
| `service_name` | string | Which service was built |
| `image_ref` | string | Full image reference (e.g., `ghcr.io/org/api`) |
| `digest` | string | `sha256:...` content digest |
| `platforms` | string[]? | e.g., `["linux/amd64", "linux/arm64"]` |
| `size_bytes` | number? | Image size |

## Build CLI

| Command | Description |
|---------|-------------|
| `eve build list [--project <id>]` | List build specs |
| `eve build show <build_id>` | Build spec details |
| `eve build create --project <id> --ref <sha> --manifest-hash <hash> [--services s1,s2] [--repo-dir <path>]` | Create a build spec |
| `eve build run <build_id>` | Start a build run |
| `eve build runs <build_id>` | List runs for a build |
| `eve build logs <build_id> [--run <id>] [--follow]` | View build logs |
| `eve build artifacts <build_id>` | List image artifacts (digests) |
| `eve build diagnose <build_id>` | Full diagnostic dump (spec + runs + artifacts + last 30 lines) |
| `eve build cancel <build_id>` | Cancel active build |

Builds happen automatically in pipeline `build` steps. Use `eve build diagnose` to debug failures.

### Build Error Classification

| Error Code | Cause | Fix |
|-----------|-------|-----|
| `auth_error` | Registry or git authentication failure | Check GHCR_USERNAME + GHCR_TOKEN secrets |
| `clone_error` | Git clone failure | Verify repo URL and GITHUB_TOKEN |
| `build_error` | Dockerfile build step failure | Check build logs for failing stage |
| `timeout_error` | Execution timeout | Increase timeout or optimize build |
| `resource_error` | Resource exhaustion (disk, memory) | Check pod resources |
| `registry_error` | Registry push failure | Verify registry auth and namespace |

## Release Model

Releases capture a deployable snapshot: git SHA + manifest hash + image digests.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | `rel_xxx` |
| `project_id` | string | Owning project |
| `git_sha` | string | 40-char commit SHA |
| `manifest_hash` | string | Hash of manifest |
| `image_digests` | object? | `{ "api": "sha256:...", "web": "sha256:..." }` |
| `build_id` | string? | Source build (if from pipeline) |
| `version` | string? | Semantic version |
| `tag` | string? | Release tag (e.g., `v1.0.0`) |

### Release CLI

```bash
eve release resolve <tag> [--project <id>]    # Resolve release by tag
```

Releases are typically created automatically by pipeline `release` steps.

## Deploy Model

Deploy requests can use a release tag or raw SHA + manifest hash.

```bash
# Deploy via pipeline (recommended)
eve env deploy staging --ref main --repo-dir ./my-app

# Deploy with specific release
eve env deploy staging --inputs '{"release_tag": "v1.0.0"}'

# Direct deploy (bypass pipeline)
eve env deploy staging --ref <sha> --direct

# Deploy with pre-built images
eve env deploy staging --ref <sha> --direct --image-tag sha-abc123
```

### Deploy Request Fields

| Field | Description |
|-------|-------------|
| `git_sha` | 40-char SHA (required unless `release_tag`) |
| `manifest_hash` | Manifest hash (required with `git_sha`) |
| `release_tag` | Resolve from existing release (alternative to sha+hash) |
| `image_digests` | Service → digest map (skips build) |
| `image_tag` | Tag for pre-built images (e.g., `local`, `sha-abc123`) |
| `direct` | Bypass pipeline, deploy directly |
| `inputs` | Additional pipeline inputs |

## Canonical Pipeline: Build → Release → Deploy

The standard deployment pipeline:

```yaml
pipelines:
  deploy:
    trigger:
      github:
        event: push
        branch: main
    steps:
      - name: build
        action: { type: build }
        # Creates BuildSpec + BuildRun, outputs build_id + image_digests

      - name: release
        depends_on: [build]
        action: { type: release }
        # Creates Release from build artifacts (digest-based image refs)

      - name: deploy
        depends_on: [release]
        action: { type: deploy, env_name: staging }
        # Deploys release to environment
```

### Promotion Pattern

1. Deploy to test → creates release with tag
2. Resolve release: `eve release resolve v1.0.0`
3. Deploy to staging: `eve env deploy staging --inputs '{"release_tag": "v1.0.0"}'`

## Build Backends

| Backend | Environment | Notes |
|---------|-------------|-------|
| BuildKit | K8s (production) | Runs as K8s job, recommended |
| Buildx | Local development | Uses local Docker |
| Kaniko | K8s (fallback) | No Docker daemon required |

## API Endpoints

```
POST /projects/{project_id}/builds              # Create build spec
GET  /projects/{project_id}/builds              # List builds
GET  /builds/{build_id}                          # Get build spec
POST /builds/{build_id}/runs                     # Start build run
GET  /builds/{build_id}/runs                     # List runs
GET  /builds/{build_id}/artifacts                # List artifacts
GET  /builds/{build_id}/logs                     # View logs
POST /builds/{build_id}/cancel                   # Cancel build

POST /projects/{project_id}/releases             # Create release
GET  /projects/{project_id}/releases             # List releases
GET  /releases/{tag}                              # Resolve by tag

POST /projects/{project_id}/environments/{env}/deploy  # Deploy
```
