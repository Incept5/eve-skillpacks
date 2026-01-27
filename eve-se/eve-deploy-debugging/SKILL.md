---
name: eve-deploy-debugging
description: Deploy and debug Eve-compatible apps via CLI, including local tag workflows, ingress URLs, and failure diagnosis. Use when deployments fail or behavior does not match expectations.
---

# Eve Deploy and Debug

Use these steps to deploy and diagnose app issues quickly.

## Environment preference

- **Staging first** for validation: `https://api.eve-staging.incept5.dev`
- Local k8s is optional for rapid iteration

## Deploy flows

- Local k8s loop:
  - `docker build -t ghcr.io/org/app-api:local ./apps/api`
  - `k3d image import ghcr.io/org/app-api:local -c eve-local`
  - `eve env deploy <project> <env> --tag local`
- Registry deploy:
  - Ensure registry secrets exist
  - `eve env deploy <project> <env>`

## Observe deploy

- `eve job list --all --phase active` to find the deploy job.
- `eve job follow <id>` for live logs.
- `eve job watch <id>` to combine status polling + logs.
- `eve job diagnose <id>` for errors and timing.
- `eve job dep list <id>` if the job is blocked.
- `eve job result <id>` once it completes.

## CLI-first debugging (CRITICAL)

| Priority | Tool | When |
|----------|------|------|
| 1st | `eve` CLI | **Always start here** |
| 2nd | `eve system health` | API + DB health |
| 3rd | `kubectl` | **Only when CLI is insufficient** |

## Common failure points

- Registry auth missing or wrong (`GHCR_USERNAME`, `GHCR_TOKEN`).
- Secrets interpolation missing (`${secret.KEY}` not set).
- Health check failing, blocking readiness.
- Environment gate held by another deploy job.

## Access URLs

- Staging ingress: `https://{component}.{project}-{env}.eve-staging.incept5.dev`.
- Local ingress: `http://{component}.{project}-{env}.lvh.me`.
- Production: `https://{component}.{project}-{env}.{domain}` (manifest domain or default domain).

## Recursive skill distillation

- Add new deploy pitfalls and fixes to this skill as they appear.
- Extract distinct flows (registry, migrations, health checks) into new skills if needed.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
