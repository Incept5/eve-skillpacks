---
name: eve-repo-upkeep
description: Keep Eve-compatible repos aligned with platform best practices and current manifest conventions.
---

# Eve Repo Upkeep

Use this workflow to keep an app repo current with Eve conventions.

## When to Use

- After Eve platform updates or manifest schema changes
- Before a major deploy or release
- When onboarding a new maintainer

## Files to Keep Current

### `.eve/manifest.yaml`

- Ensure `schema: eve/compose/v1` is present.
- Prefer `services:` over legacy `components:`.
- Keep `x-eve` ingress and pipeline definitions accurate.
- Keep `x-eve.defaults` in sync with harness defaults (harness/profile/options).
- Keep `x-eve.agents` profiles aligned with orchestration policy.
- Confirm `${secret.KEY}` usage for secrets.
- Deploy pipelines should include a `build` step before `release`.
- Services with Docker images should have `build.context` defined.
- Registry auth secrets (GHCR_USERNAME, GHCR_TOKEN or GITHUB_TOKEN) should be configured.

### `skills.txt`

- Keep Eve skillpack references up to date.
- Remove obsolete packs or pinned versions.

### Agent Instructions (AGENTS.md / CLAUDE.md)

- Update skill references to include `eve-se-index`.
- Remove stale commands or URLs.

## Check for Deprecated Patterns

- Old CLI commands (`eve deploy` vs `eve env deploy`)
- Old deploy syntax without `--ref` parameter
- Hardcoded domains in docs or manifests
- Inline secrets in repo files
- Dockerfiles missing `org.opencontainers.image.source` label pointing to the repo URL
- Pipelines missing `build` step before `release`
- Services with Docker images but no `build.context` configuration
- Missing registry authentication secrets (GHCR_USERNAME, GHCR_TOKEN/GITHUB_TOKEN)

## Test After Updates

```bash
# Local validation (Docker Compose)
docker compose up --build

# Staging deploy (requires --ref with git SHA or branch name)
eve env deploy staging --ref main

# Use --direct to bypass pipeline if needed
eve env deploy staging --ref main --direct
```

Track the deploy job with `eve job follow`.
