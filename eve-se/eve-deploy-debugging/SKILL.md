---
name: eve-deploy-debugging
description: Deploy and debug Eve-compatible apps via the CLI, with a focus on staging environments.
---

# Eve Deploy and Debug

Use these steps to deploy and diagnose app issues quickly.

## Environment Setup

- Get the staging API URL from your admin.
- Create and use a profile:

```bash
eve profile create staging --api-url https://api.eh1.incept5.dev
eve profile use staging
```

## Deploy Flow (Staging)

```bash
# Create env if needed
eve env create staging --project proj_xxx --type persistent

# Deploy (requires --ref with 40-char SHA or a ref resolved against --repo-dir)
eve env deploy staging --ref main --repo-dir .

# When environment has a pipeline configured, the above triggers the pipeline.
# Use --direct to bypass pipeline and deploy directly:
eve env deploy staging --ref main --repo-dir . --direct

# Pass inputs to pipeline:
eve env deploy staging --ref main --repo-dir . --inputs '{"key":"value"}'
```

## Observe the Deploy

```bash
eve job list --phase active
eve job follow <job-id>
eve job watch <job-id>
eve job diagnose <job-id>
eve job result <job-id>
```

## CLI-First Debugging

1. `eve job follow` and `eve job diagnose` for the deploy job
2. `eve system health` for API + DB health

Only use cluster tools if you have access and the CLI is insufficient.

## Common Failure Points

- Registry auth missing (`GHCR_USERNAME`, `GHCR_TOKEN`)
- Secrets interpolation missing (`${secret.KEY}` not set)
- Healthcheck failing, blocking readiness
- Environment gate held by another deploy job

### Build Failures

If a deploy pipeline fails at the build step:

```bash
# Find the build ID from the pipeline run
eve build list --project <project_id>

# Full diagnostic
eve build diagnose <build_id>

# Check build logs for errors
eve build logs <build_id>

# Verify registry credentials are set
eve secrets list --project <project_id>
# Required: GHCR_USERNAME, GHCR_TOKEN (or GITHUB_TOKEN)
```

Common build failures:
- **Registry auth**: Missing GHCR_USERNAME or GHCR_TOKEN secrets
- **Dockerfile not found**: Check `build.context` path in manifest
- **Multi-stage build failure**: BuildKit handles these correctly; Kaniko may have issues
- **Workspace errors**: Build context not available — check `eve build diagnose`

## Access URLs

- URL pattern: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}`
- Ask the admin for the correct domain (staging vs production).

## Related Skills

- Local dev loop: `eve-local-dev-loop`
- Secrets: `eve-auth-and-secrets`
- Manifest changes: `eve-manifest-authoring`
