---
name: eve-troubleshooting
description: Troubleshoot common Eve deploy and job failures using CLI-first diagnostics.
---

# Eve Troubleshooting

Use CLI-first diagnostics. Do not assume cluster access.

## Quick Triage Checklist

```bash
eve system health
eve auth status
eve job list --phase active
```

## Common Issues and Fixes

### Auth Fails or "Not authenticated"

```bash
eve auth logout
eve auth login
eve auth status
```

If SSH key is missing, register it with the admin or follow the CLI prompt to fetch from GitHub.

### Secret Missing / Interpolation Error

```bash
eve secrets list --project proj_xxx
eve secrets set MISSING_KEY "value" --project proj_xxx
```

Verify `.eve/dev-secrets.yaml` exists for local interpolation.

### Deploy Job Failed

```bash
eve job follow <job-id>
eve job diagnose <job-id>
eve job result <job-id>
eve env diagnose <project> <env>     # structured DeployFailure + cluster snapshot
```

Check for registry auth errors, missing secrets, or healthcheck failures.

`eve env diagnose` surfaces a typed `last_deploy_failure` (kind, service, pod, namespace, message) plus the live K8s state. Read the failure `kind` first — the CLI no longer collapses real causes into a bare `HTTP request failed`. Compare the diagnose `manifest_hash` against the latest sync to spot applied-release drift.

### Sentinel Slack Alerts

Slack pings from the platform sentinel point at a degraded env. Don't act on the alert text alone — pull the project/env out of it and re-confirm with `eve env diagnose <project> <env>`. If the env reads healthy, the alert was a transient that has self-healed; if not, follow the standard deploy-failure triage above.

### Custom Domain Issues

```bash
eve domain verify <hostname>     # DNS check + cert state + next steps
eve domain status <hostname>     # which env owns it
eve domain list --env <env>      # everything bound to this env
```

Common causes:
- **DNS not resolved**: `verify` will print the expected target and `dns_result.verified: false` — fix the CNAME/A record before re-running deploy.
- **Cert pending**: cert-manager HTTP-01 challenge in flight; re-run `verify` after a minute.
- **First-bind-wins conflict**: another env already claimed the hostname. Use `eve domain transfer <host> --to <env>` and redeploy, or scope the domain per-env via `environments.<env>.overrides`.

### Registry Push Fails with UNAUTHORIZED

If build jobs fail with `UNAUTHORIZED: authentication required` when pushing:

1. Verify secrets are set: `eve secrets list --project proj_xxx`
2. If using a custom BYO registry, verify credentials map to `registry.host`
3. Confirm the imagePull metadata in your manifest is correct
4. Add OCI source label to Dockerfile: `LABEL org.opencontainers.image.source="https://github.com/ORG/REPO"`

Some registries require repository-linked package metadata or workspace-level auth alignment.

### Build Failures

#### Symptoms
- Pipeline fails at build step
- `eve build diagnose` shows run status = `failed`

#### Triage
```bash
eve build list --project <id>          # Find recent builds
eve build diagnose <build_id>          # Full state dump
eve build logs <build_id>              # Raw build output
```

#### Common Causes

**Registry authentication:**
- If using custom registry mode, verify `REGISTRY_USERNAME` and `REGISTRY_PASSWORD` secrets are set (or provider-equivalent registry credentials). With managed registry (`registry: "eve"`), this step is usually not required.
- Ensure credentials can access the configured registry account and namespace
- Check: `eve secrets list --project <id>`

**Dockerfile issues:**
- Service must have `build.context` in manifest pointing to directory with Dockerfile
- Dockerfile path defaults to `<context>/Dockerfile`
- Multi-stage builds work with BuildKit; may fail with Kaniko

**Workspace/clone errors:**
- Build requires workspace at the correct git SHA
- Check `eve build diagnose` for workspace preparation errors

**Image push failures:**
- OCI labels help link packages to repos: add `LABEL org.opencontainers.image.source="https://github.com/OWNER/REPO"` to Dockerfile
- Ensure registry host and auth match manifest `registry.host` when using BYO/custom registry

### Job Stuck or Blocked

```bash
eve job show <job-id>
eve job dep list <job-id>
```

Resolve dependencies or update phase with `eve job update` if appropriate.

Stale-attempt recovery now covers all assignees and graceful agent-runtime shutdown, and agent-runtime pod death no longer strands attempts. If a job still looks stuck after a recent restart, give the recovery loop one cycle before manually intervening — it usually reclaims on its own.

### Chat Route Doesn't Match

Chat route matching is now case-insensitive on the regex side. If a route still fails to fire, the pattern itself is the suspect — not the casing. Re-check the `match` regex in your chat config, then resync the manifest.

### Agent-Runtime Org Discovery

Agent-runtime auto-discovers orgs at startup; you no longer need an `org_default` config entry. If a fresh deploy can't see an org's jobs, check that the org actually has at least one project + agent registered, then check agent-runtime logs — don't go looking for a missing `org_default`.

### App Not Reachable After Deploy

- Confirm deploy job succeeded (`eve job result`).
- Validate ingress host pattern: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}`.
- Ensure service port matches `x-eve.ingress.port`.

## Escalation

If CLI output is insufficient, collect:

- `eve system health`
- `eve job diagnose <job-id>`
- manifest diff (recent changes)

Then hand off to the platform operator.
