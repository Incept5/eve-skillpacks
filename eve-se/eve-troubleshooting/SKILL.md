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

Verify `.eve/secrets.yaml` exists for local interpolation.

### Deploy Job Failed

```bash
eve job follow <job-id>
eve job diagnose <job-id>
eve job result <job-id>
```

Check for registry auth errors, missing secrets, or healthcheck failures.

### Job Stuck or Blocked

```bash
eve job show <job-id>
eve job dep list <job-id>
```

Resolve dependencies or update phase with `eve job update` if appropriate.

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
