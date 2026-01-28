---
name: eve-pipelines-workflows
description: Define and run Eve pipelines and workflows via manifest and CLI. Use when wiring build, release, deploy flows or invoking workflow jobs.
---

# Eve Pipelines and Workflows

Use these patterns to automate build and deploy actions and invoke workflow jobs.

## Pipelines (v2 steps)

- Define pipelines under `pipelines` in `.eve/manifest.yaml`.
- Steps can be `action`, `script`, or `agent`.
- Use `depends_on` to control ordering.
- Built-in actions include `build`, `release`, `deploy`, `run`, `job`, `create-pr`.
- Run manually:
  - `eve pipeline list`
  - `eve pipeline show <project> <name>`
  - `eve pipeline run <name> --ref <sha> --env <env>`
- Trigger blocks exist in the manifest; GitHub and Slack webhooks can create pipeline runs.

Track pipeline execution like any job:

```bash
eve job list --phase active
eve job follow <job-id>
eve job result <job-id>
```

## Environment Deploy as Pipeline Alias

When an environment has a `pipeline` configured in the manifest, `eve env deploy <env> --ref <sha>` automatically triggers that pipeline instead of doing a direct deploy.

### Basic usage

```bash
# Triggers the configured pipeline for test environment
eve env deploy test --ref abc123

# Pass inputs to the pipeline
eve env deploy staging --ref abc123 --inputs '{"release_id":"rel_xxx"}'

# Bypass pipeline and do direct deploy
eve env deploy staging --ref abc123 --direct
```

### Promotion flow example

```bash
# 1. Build and deploy to test environment
eve env deploy test --ref abc123

# 2. Get release info from the test build
eve release resolve v1.2.3
# Output: rel_xxx

# 3. Promote to staging using the release_id
eve env deploy staging --ref abc123 --inputs '{"release_id":"rel_xxx"}'
```

### Key behaviors

- If `environments.<env>.pipeline` is set, `eve env deploy <env>` triggers that pipeline
- Use `--direct` flag to bypass the pipeline and perform a direct deploy
- Use `--inputs '{"key":"value"}'` to pass inputs to the pipeline run
- Default inputs can be configured via `environments.<env>.pipeline_inputs` in the manifest
- The `--ref` flag specifies which git SHA to deploy
- Environment variables and secrets are interpolated as usual

This pattern enables promotion workflows where you build once in a lower environment and promote the same artifact through higher environments.

## Workflows

- Define workflows under `workflows` in the manifest.
- `db_access` is honored when present (`read_only`, `read_write`).
- Invoke manually:
  - `eve workflow list`
  - `eve workflow show <project> <name>`
  - `eve workflow run <project> <name> --input '{"k":"v"}'`
- Invocation creates a job; track it with normal job commands.

## Recursive skill distillation

- Add new pipeline or workflow patterns as they emerge.
- Split specialized flows into new eve-se skills when needed.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
