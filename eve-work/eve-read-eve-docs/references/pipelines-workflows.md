# Pipelines + Workflows (Current)

## Pipelines (Manifest)

Pipelines are ordered steps that expand into a job graph.

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

### Step Types

- **action**: built-in actions (`build`, `release`, `deploy`, `run`, `job`, `create-pr`, `env-ensure`)
- **script**: shell command executed by worker (`run` + `timeout`)
- **agent**: AI agent job (prompt-driven)
- **run**: shorthand for `script.run`

### Pipeline Runs

- A run creates one job per step with dependencies wired from `depends_on`.
- Run IDs: `prun_xxx`.
- `eve pipeline run --only <step>` uses the job-graph expander.

### CLI

```bash
eve pipeline list [project]
eve pipeline show <project> <name>
eve pipeline run <name> --ref <sha> --env <env> --inputs '{"k":"v"}'
eve pipeline runs [project] --status <status>
eve pipeline show-run <pipeline> <run-id>
eve pipeline approve <run-id>
eve pipeline cancel <run-id>
eve pipeline logs <pipeline> <run-id> --step <name>
```

### Env Deploy as Pipeline Alias

If `environments.<env>.pipeline` is set, `eve env deploy <env> --ref <sha>` triggers the pipeline.
Use `--direct` to bypass.

### Promotion Pattern

1) Deploy to test (creates release)
2) Resolve release (`eve release resolve vX.Y.Z`)
3) Deploy to staging/production with `--inputs '{"release_id":"rel_xxx"}'`

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

### Invocation

- Invoking a workflow creates a **job** with workflow metadata in `hints`.
- `wait=true` returns `result_json` with a 60s timeout.

### CLI

```bash
eve workflow list [project]
eve workflow show <project> <name>
eve workflow run <project> <name> --input '{"k":"v"}'
eve workflow invoke <project> <name> --input '{"k":"v"}'
eve workflow logs <job-id>
```

## Triggers

Both pipelines and workflows can include a `trigger` block. The orchestrator matches incoming events and creates pipeline runs or workflow jobs.

### GitHub Trigger Examples

```yaml
trigger:
  github:
    event: pull_request
    action: [opened, synchronize]
    base_branch: main
```

Supported actions include `opened`, `synchronize`, `reopened`, `closed`.
Branch patterns support wildcards like `release/*` or `*-prod`.

## Planned (Not Implemented)

- Workflow schema validation (inputs/outputs).
- Skill-based workflows with workflow-specific SKILL.md metadata.
