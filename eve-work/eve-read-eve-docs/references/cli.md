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
eve profile set --default-email you@example.com

eve profile set --org org_xxx --project proj_xxx
```

## Auth

```bash
eve auth status
eve auth login --email you@example.com
eve auth login --email you@example.com --ttl 30  # custom token TTL (1-90 days)
eve auth permissions

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

eve admin invite --email user@example.com --github user
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

# Membership management
eve org members --org org_xxx
eve org members add user@example.com --role admin --org org_xxx
eve org members remove user_abc --org org_xxx

eve project members --project proj_xxx
eve project members add user@example.com --role admin --project proj_xxx
eve project members remove user_abc --project proj_xxx
```

**URL impact:** The org `--slug` and project `--slug` directly form deployment URLs and K8s namespaces:
- URL: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}` (e.g., `api.myorg-myproj-staging.eh1.incept5.dev`)
- Namespace: `eve-{orgSlug}-{projectSlug}-{env}` (e.g., `eve-myorg-myproj-staging`)
- `${ORG_SLUG}` is available for interpolation in manifest values (see `references/manifest.md`)

Slugs are immutable after creation. Choose short, meaningful values.

`repo_url` accepts HTTPS, SSH (`git@host:org/repo.git`), or `file://` (local/dev only).

## Jobs (see `references/jobs.md` for full detail)

```bash
eve job create --description "Fix bug"
eve job list --phase active
eve job show <job-id>
eve job follow <job-id>
eve job diagnose <job-id>
```

## Builds

Builds are first-class primitives tracking image construction.

| Command | Description |
|---------|-------------|
| `eve build list [--project <id>]` | List build specs |
| `eve build show <build_id>` | Build spec details |
| `eve build create --project <id> --ref <sha> --manifest-hash <hash> [--services <list>] [--repo-dir <path>]` | Create a build spec |
| `eve build run <build_id>` | Start a build run |
| `eve build runs <build_id>` | List runs for a build |
| `eve build logs <build_id> [--run <id>]` | View build logs |
| `eve build artifacts <build_id>` | List image artifacts (digests) |
| `eve build diagnose <build_id>` | Full diagnostic dump |
| `eve build cancel <build_id>` | Cancel active build |

Builds happen automatically in pipeline `build` steps. Use `eve build diagnose` to debug.

## Pipelines

```bash
eve pipeline list
eve pipeline show <project> <name>
eve pipeline run <name> --ref 0123456789abcdef0123456789abcdef01234567 --env staging --inputs '{"k":"v"}'
eve pipeline run <name> --ref main --repo-dir ./my-app --env staging
eve pipeline approve <run-id>
eve pipeline cancel <run-id>
```

## Workflows

```bash
eve workflow list
eve workflow show <project> <name>
eve workflow run <project> <name> --input '{"k":"v"}'
eve workflow invoke <project> <name> --input '{"k":"v"}'
eve workflow logs <job-id>
```

## Deployments

```bash
eve env deploy staging --ref main --repo-dir ./my-app

eve env show <project> staging
eve env diagnose <project> staging
eve env logs <project> staging
eve env delete <project> staging
```

If an environment has a pipeline configured, `eve env deploy` triggers that pipeline.
Use `--direct` to bypass the pipeline. `--ref` must be a 40-character SHA, or a ref
resolved against `--repo-dir`/cwd.

## Secrets

```bash
eve secrets list --project proj_xxx
eve secrets set KEY value --project proj_xxx

eve secrets import --org org_xxx --file ./secrets.env

eve secrets validate --project proj_xxx
```

## Chat + Integrations (Slack)

```bash
eve integrations list --org org_xxx
eve integrations slack connect --org org_xxx --team-id T123 --token xoxb-test
eve integrations test <integration_id> --org org_xxx

# Default agent slug (org-wide fallback)
eve org update org_xxx --default-agent mission-control

# Simulate inbound chat (project-scoped)
eve chat simulate --project proj_xxx --team-id T123 --channel-id C123 --user-id U123 --text "hello"
```

Slack commands (run inside Slack):

```text
@eve <agent-slug> <command>
@eve agents list
@eve agents listen <agent-slug>
@eve agents unlisten <agent-slug>
@eve agents listening
```

## Events (Triggers)

```bash
eve event list --project proj_xxx --type github.push
eve event emit --type manual.test --source manual --payload '{"k":"v"}'
```

## Debugging (CLI-first)

See `references/deploy-debug.md` for the debugging ladder and system health commands.

## System (Internal)

```bash
eve system orchestrator status
eve system orchestrator set-concurrency <n>
eve system status
eve system jobs
eve system envs
eve system logs api --tail 50
eve system pods
eve system events
```
