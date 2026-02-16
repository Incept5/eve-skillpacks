# Pipelines + Workflows (Current)

## Pipelines (Manifest)

Pipelines are ordered steps that expand into a job graph. Define them in `.eve/manifest.yaml`.

```yaml
pipelines:
  deploy-test:
    trigger:
      github:
        event: push
        branch: main
    steps:
      - name: build
        action: { type: build }
      - name: unit-tests
        script: { run: "pnpm test", timeout: 1800 }
      - name: deploy
        depends_on: [build, unit-tests]
        action: { type: deploy }
```

### Canonical Pipeline Pattern

The standard build-release-deploy pipeline:

```yaml
steps:
  - name: build
    action: { type: build }
    # Creates BuildSpec + BuildRun, outputs build_id + image_digests
  - name: release
    depends_on: [build]
    action: { type: release }
    # References build_id, uses digest-based image refs from BuildArtifacts
  - name: deploy
    depends_on: [release]
    action: { type: deploy, env_name: staging }
```

### Step Output Linking

Understand how data flows between pipeline steps:

- The `build` action creates BuildSpec and BuildRun records. On success, it emits `build_id` and `image_digests` as step outputs.
- BuildRuns produce BuildArtifacts containing per-service image digests (`sha256:...`).
- The `release` action automatically receives `build_id` from the upstream build step. It derives `image_digests_json` from BuildArtifacts, ensuring immutable digest-based image references.
- The `deploy` action references images by digest for deterministic, reproducible deployments.

This chain ensures that what was built is exactly what gets released and deployed -- no tag mutation, no ambiguity.

### Step Types

- **action**: built-in actions (`build`, `release`, `deploy`, `run`, `job`, `create-pr`, `notify`, `env-ensure`, `env-delete`)
- **script**: shell command executed by worker (`run` or `command` + `timeout`)
- **agent**: AI agent job (prompt-driven)
- **run**: shorthand for `script.run`

### Pipeline Runs

- A run creates one job per step with dependencies wired from `depends_on`.
- Run IDs: `prun_xxx`.
- Pipeline runs use the job-graph expander by default.
- `eve pipeline run --only <step>` runs a subset of steps.
- A failed job marks the run as failed and cascades cancellation to dependents.
- Cancelled jobs are terminal and unblock downstream jobs.

### CLI

```bash
eve pipeline list [project]
eve pipeline show <project> <name>
eve pipeline run <name> --ref <sha> --env <env> --inputs '{"k":"v"}' --repo-dir ./my-app
eve pipeline runs [project] --status <status>
eve pipeline show-run <pipeline> <run-id>
eve pipeline approve <run-id>
eve pipeline cancel <run-id> [--reason <text>]
eve pipeline logs <pipeline> <run-id> [--step <name>]
```

Notes:
- `--ref` must be a 40-character SHA, or a ref resolved against `--repo-dir`/cwd.

### Env Deploy as Pipeline Alias

If `environments.<env>.pipeline` is set, `eve env deploy <env> --ref <sha>` triggers the pipeline.
Use `--direct` to bypass. `--ref` must be a 40-character SHA, or a ref resolved
against `--repo-dir`/cwd.

### Promotion Pattern

1. Deploy to test (creates release):
   `eve env deploy test --ref <sha>`
2. Resolve release:
   `eve release resolve vX.Y.Z`
3. Deploy to staging/production with:
   `eve env deploy staging --ref <sha> --inputs '{"release_id":"rel_xxx"}'`

This enables build-once, deploy-many promotion workflows without rebuilding images.

## Pipeline Logs and Streaming

### Snapshot Logs

View build and execution logs (not just metadata) with timestamps and step name prefixes:

```bash
eve pipeline logs <pipeline> <run-id>                  # All step logs
eve pipeline logs <pipeline> <run-id> --step <name>    # Single step
```

### Live Streaming

Stream logs in real time via SSE:

```bash
eve pipeline logs <pipeline> <run-id> --follow                   # All steps
eve pipeline logs <pipeline> <run-id> --follow --step <name>     # Single step
```

Output format:

```
[14:23:07] [build] Cloning repository...
[14:23:09] [build] buildkit addr: tcp://buildkitd.eve.svc:1234
[14:23:15] [build] [api] #5 [dependencies 1/4] COPY pnpm-lock.yaml ...
[14:24:01] [deploy] Deployment started; waiting up to 180s
[14:24:12] [deploy] Deployment status: 1/1 ready
```

### Failure Hints

When a build step fails, the CLI automatically shows:
- The error type and classification
- An actionable hint (e.g., `Run 'eve build diagnose bld_xxx'`)
- The build ID for cross-referencing

### Pipeline-to-Build Linkage

Pipeline steps of type `build` create build specs and runs. On failure:
1. The pipeline step error includes the build ID.
2. The CLI prints a hint to run `eve build diagnose <build_id>`.
3. Build diagnosis shows the full buildkit output and the failed Dockerfile stage.

## Workflow Definitions

Workflows are defined in the manifest and invoked as jobs.

```yaml
workflows:
  nightly-audit:
    db_access: read_only
    hints:
      gates: ["remediate:proj_xxx:staging"]
    steps:
      - agent:
          prompt: "Audit error logs and summarize anomalies"
```

### Workflow Hints

Workflow definitions may include a `hints` block. These hints are merged into the job at invocation time (API, CLI, or event triggers). Use hints for:

- **Remediation gates**: control which environments a workflow can remediate. Pattern: one gate per environment.
  ```yaml
  hints:
    gates: ["remediate:proj_abc123:staging"]
  ```
- **Timeouts**: set execution time limits for the workflow job.
- **Harness preferences**: specify model/harness settings that override project defaults for this workflow.

### Invocation

- Invoking a workflow creates a **job** with workflow metadata in `hints`.
- `wait=true` returns `result_json` with a 60s timeout.

### Workflow CLI

```bash
eve workflow list [project]
eve workflow show <project> <name>
eve workflow run <project> <name> --input '{"k":"v"}'
eve workflow invoke <project> <name> --input '{"k":"v"}'
eve workflow logs <job-id>
```

## Triggers

Both pipelines and workflows can include a `trigger` block. The orchestrator matches incoming events and creates pipeline runs or workflow jobs.

### GitHub Push Triggers

```yaml
trigger:
  github:
    event: push
    branch: main
```

Branch patterns support wildcards (e.g., `release/*`, `*-prod`).

### GitHub Pull Request Triggers

```yaml
trigger:
  github:
    event: pull_request
    action: [opened, synchronize]
    base_branch: main
```

Supported PR actions: `opened`, `synchronize`, `reopened`, `closed`.
Base branch filtering supports wildcard patterns.

### PR Preview Deployment Example

Deploy a preview environment on PR open/update, clean up on close:

```yaml
pipelines:
  pr-preview:
    trigger:
      github:
        event: pull_request
        action: [opened, synchronize]
        base_branch: main
    steps:
      - name: create-preview-env
        action:
          type: env-ensure
          with:
            env_name: ${{ env.pr_${{ github.pull_request.number }} }}
            kind: preview
      - name: deploy
        depends_on: [create-preview-env]
        action:
          type: deploy
          with:
            env_name: ${{ env.pr_${{ github.pull_request.number }} }}

  pr-cleanup:
    trigger:
      github:
        event: pull_request
        action: closed
        base_branch: main
    steps:
      - name: cleanup-env
        action:
          type: env-delete
          with:
            env_name: ${{ env.pr_${{ github.pull_request.number }} }}
```

## API Endpoints

```
GET  /projects/{project_id}/pipelines
GET  /projects/{project_id}/pipelines/{name}

# Pipeline runs
POST /projects/{project_id}/pipelines/{name}/run
GET  /projects/{project_id}/pipelines/{name}/runs
GET  /projects/{project_id}/pipelines/{name}/runs/{run_id}
POST /pipeline-runs/{run_id}/approve
POST /pipeline-runs/{run_id}/cancel
GET  /pipeline-runs/{run_id}/stream
GET  /pipeline-runs/{run_id}/steps/{name}/stream

# Workflows
GET  /projects/{project_id}/workflows
GET  /projects/{project_id}/workflows/{name}
POST /projects/{project_id}/workflows/{name}/invoke?wait=true|false
```
