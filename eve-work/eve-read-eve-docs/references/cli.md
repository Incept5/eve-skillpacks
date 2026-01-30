# CLI Quick Reference (Current)

## Default Environment (Staging)

Default to **staging** for user guidance. Use local/docker only when explicitly asked to do local development.
Use the staging API URL: https://api.eh1.incept5.dev

## One Required Env Var

Set the API endpoint and the CLI works everywhere:

```bash
export EVE_API_URL=https://api.eh1.incept5.dev   # staging (default)
export EVE_API_URL=http://localhost:4801               # local/docker (opt-in)
export EVE_API_URL=http://api.eve.lvh.me               # k8s ingress (local)
```

If you have access to `./bin/eh status`, use it to get the correct URL for the current instance. For staging, use https://api.eh1.incept5.dev.

## Profiles and Config

```bash
eve profile create staging --api-url https://api.eh1.incept5.dev
eve profile show
eve config set --default-email you@example.com

eve profile set --org org_xxx --project proj_xxx
```

## Auth

```bash
eve auth status
eve auth login --email you@example.com

# Check local AI tool credentials (Claude Code, Codex)
eve auth creds
eve auth creds --claude             # Only check Claude
eve auth creds --json               # JSON output

# Sync local OAuth tokens to Eve
eve auth sync                       # Sync to user-level (default)
eve auth sync --org org_xxx         # Sync to org-level
eve auth sync --project proj_xxx    # Sync to project-level
eve auth sync --dry-run             # Preview without syncing

eve auth bootstrap --email you@example.com --token $EVE_BOOTSTRAP_TOKEN

eve admin invite --email user@example.com --github-username user
```

Notes:
- CLI can auto-fetch SSH keys from GitHub when none are registered.
- `auth creds` shows what Claude/Codex credentials are available locally.
- `auth sync` pushes local OAuth tokens to Eve (defaults to user-level).

## Org / Project

```bash
eve org list
eve org ensure "my-org" --slug myorg

eve project list
eve project ensure --name "My Project" --slug myproj \
  --repo-url https://github.com/org/repo --branch main

eve project show proj_xxx
eve project sync
```

**URL impact:** The org `--slug` and project `--slug` directly form deployment URLs and K8s namespaces:
- URL: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}` (e.g., `api.myorg-myproj-staging.eh1.incept5.dev`)
- Namespace: `eve-{orgSlug}-{projectSlug}-{env}` (e.g., `eve-myorg-myproj-staging`)
- `${ORG_SLUG}` is available for interpolation in manifest values (see `references/manifest.md`)

Slugs are immutable after creation. Choose short, meaningful values.

## Jobs (see `references/jobs.md` for full detail)

```bash
eve job create --description "Fix bug"
eve job list --phase active
eve job show <job-id>
eve job follow <job-id>
eve job diagnose <job-id>
```

## Pipelines

```bash
eve pipeline list
eve pipeline show <project> <name>
eve pipeline run <name> --ref <sha> --env staging --inputs '{"k":"v"}'
eve pipeline approve <run-id>
eve pipeline cancel <run-id>
```

## Workflows

```bash
eve workflow list
eve workflow show <project> <name>
eve workflow run <project> <name> --input '{"k":"v"}'
```

## Deployments

```bash
eve env deploy staging --ref main

eve env status staging
```

If an environment has a pipeline configured, `eve env deploy` triggers that pipeline.
Use `--direct` to bypass the pipeline.

## Secrets

```bash
eve secrets list --project proj_xxx
eve secrets set KEY value --project proj_xxx

eve secrets import --org org_xxx --file ./secrets.env

eve secrets validate --project proj_xxx
```

## Events (Triggers)

```bash
eve event list --project proj_xxx --type github.push
eve event emit --type manual.test --source manual --payload '{"k":"v"}'
```

## Debugging (CLI-first)

See `references/deploy-debug.md` for the debugging ladder and system health commands.
