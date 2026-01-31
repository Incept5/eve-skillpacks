---
name: eve-cli-primitives
description: Core Eve CLI primitives and capabilities for app developers. Use as the quick reference for commands and flows.
---

# Eve CLI Primitives

Use this skill as the command map for Eve. Keep examples short and concrete.

## Profiles (API Target + Defaults)

```bash
# Create and use a profile
eve profile create staging --api-url https://api.eh1.incept5.dev
eve profile use staging

# Set defaults to avoid repeating flags
eve profile set --default-email you@example.com --default-ssh-key ~/.ssh/id_ed25519
eve profile set --org org_xxx --project proj_xxx

# Inspect current profile
eve profile show
```

## Authentication

```bash
eve auth login
eve auth status
eve auth logout

# Sync local OAuth tokens for agent harnesses (optional)
eve auth sync
```

## Orgs and Projects

```bash
eve org list
eve org ensure my-org --slug myorg

eve project list
eve project ensure --name "My App" --slug my-app --repo-url git@github.com:me/my-app.git --branch main
```

**URL impact:** The org `--slug` and project `--slug` directly form your deployment URLs and K8s namespaces:
- URL: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}` (e.g., `api.myorg-my-app-staging.eh1.incept5.dev`)
- Namespace: `eve-{orgSlug}-{projectSlug}-{env}` (e.g., `eve-myorg-my-app-staging`)

Choose slugs carefully — they are immutable after creation.

## Environments and Deploys

```bash
# Create a persistent environment
eve env create staging --project proj_xxx --type persistent

# Inspect environments
eve env list --project proj_xxx
eve env show staging --project proj_xxx
eve env status staging --project proj_xxx

# Deploy an environment (requires --ref with git SHA or branch name)
eve env deploy staging --ref main

# When environment has a pipeline configured, the above triggers the pipeline.
# Use --direct to bypass pipeline and deploy directly:
eve env deploy staging --ref main --direct

# Pass inputs to pipeline:
eve env deploy staging --ref main --inputs '{"key":"value"}'
```

## Jobs (Create + Observe)

```bash
eve job create --description "Review auth flow"
eve job list --phase active
eve job show <job-id>
eve job follow <job-id>
eve job diagnose <job-id>
eve job result <job-id>
```

## Secrets

```bash
eve secrets list --project proj_xxx
eve secrets set API_KEY "value" --project proj_xxx
eve secrets delete API_KEY --project proj_xxx
```

## Pipelines and Workflows

```bash
eve pipeline list
eve pipeline show <project> <name>
eve pipeline run <name> --ref <sha> --env <env>

eve workflow list
eve workflow show <project> <name>
eve workflow run <project> <name> --input '{"k":"v"}'
```

## Builds

Builds are first-class primitives that track image construction from input (spec) to execution (run) to output (artifacts).

```bash
# List builds for a project
eve build list [--project <id>]

# Show build spec details
eve build show <build_id>

# Start a build run
eve build run <build_id>

# List runs for a build
eve build runs <build_id>

# View build logs
eve build logs <build_id> [--run <run_id>]

# List produced image artifacts (digests)
eve build artifacts <build_id>

# Full diagnostic dump (spec + runs + artifacts + logs)
eve build diagnose <build_id>

# Cancel an active build
eve build cancel <build_id>
```

Builds happen automatically during pipeline `build` steps. Use `eve build diagnose` to debug build failures.

## System Health

```bash
eve system health
```

## Harnesses (Optional)

```bash
eve harness list
eve harness list --capabilities
eve harness get mclaude
eve agents config --json
```

## Notes

- Use `--project` if no default project is set in the profile.
- Prefer `eve job ...` (singular) for job commands.
