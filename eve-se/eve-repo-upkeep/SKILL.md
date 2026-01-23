---
name: eve-repo-upkeep
description: Keep Eve-compatible repos aligned with platform best practices, including starter template updates and documentation maintenance
---

# Eve Repo Upkeep

Use this workflow to keep Eve-compatible repos current with platform changes and best practices.

## When to use

- After Eve platform updates or schema changes.
- When maintaining internal Eve projects.
- Periodically to catch deprecated patterns or configuration drift.
- Before major deploys or releases.

## Files to keep current

### .eve/manifest.yaml

- Verify schema matches the latest version.
- Check for new fields: `healthcheck`, `migrations`, `depends_on`, workflow triggers.
- Ensure secret interpolation uses `${secret.KEY}` format.
- Validate pipeline and workflow structure against current syntax.

### skills.txt

- Reference latest skillpack versions or commit hashes.
- Remove obsolete skillpacks.
- Add new eve-se skills as they become available.

### .eve/hooks/on-clone.sh

- Make idempotent (check before install).
- Use `claude mcp install` instead of deprecated methods.
- Test with a fresh clone to ensure hook executes cleanly.

### CLAUDE.md / AGENTS.md

- Update agent instructions to reference current skills and workflows.
- Remove stale instructions or deprecated commands.
- Align with project's current architecture and conventions.

## Syncing with starter template

For internal Eve repos:

- Compare against `eve-horizon-starter` or the canonical template.
- Pull in structural improvements (hooks, manifest patterns, CI config).
- Adapt rather than copy; preserve project-specific customizations.

## Checking for deprecated patterns

- Search for old CLI commands (`eve deploy` vs `eve env deploy`).
- Look for hardcoded URLs or domains that should be manifest-driven.
- Check for inline secrets that belong in `.eve/secrets.yaml` or the API.
- Verify build contexts and Dockerfiles match current conventions.

## Testing after updates

- Run a full deploy cycle: `eve env deploy <project> <env>`.
- Use `eve job follow` and `eve job diagnose` to catch issues early.
- Access ingress URLs to verify runtime behavior.
- Test local k8s flow if the repo supports it.

## Recursive skill distillation

- Record recurring upkeep patterns as updates to this skill.
- If a specific area (manifest upgrades, hook maintenance) grows complex, extract it into a new eve-se skill.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
