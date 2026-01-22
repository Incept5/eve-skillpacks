---
name: eve-pipelines-workflows
description: Define and run Eve pipelines and workflows via manifest and CLI. Use when wiring build, release, deploy flows or invoking workflow jobs.
---

# Eve Pipelines and Workflows

Use these patterns to automate build and deploy actions and invoke workflow jobs.

## Pipelines

- Define pipelines under `pipelines` in `.eve/manifest.yaml`.
- Steps can be `action`, `script`, or `agent`.
- Use `depends_on` to control ordering.
- Run manually:
  - `eve pipeline list`
  - `eve pipeline show <project> <name>`
  - `eve pipeline run <name> --ref <sha> --env <env>`
- Trigger blocks exist in the manifest; automatic trigger execution is planned.

## Workflows

- Define workflows under `workflows` in the manifest.
- Invoke manually:
  - `eve workflow list`
  - `eve workflow show <project> <name>`
  - `eve workflow run <project> <name> --input '{"k":"v"}'`
- Invocation creates a job; track it with normal job commands.

## Recursive skill distillation

- Add new pipeline or workflow patterns as they emerge.
- Split specialized flows into new eve-se skills when needed.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
