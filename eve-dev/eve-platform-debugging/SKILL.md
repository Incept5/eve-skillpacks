---
name: eve-platform-debugging
description: Debug Eve Horizon platform failures and job execution issues across API, orchestrator, worker, and runner. Use when jobs are stuck, failing, or system health is unclear in the core platform.
---

# Eve Platform Debugging

## CLI-First Philosophy

> **Always debug via the Eve CLI first.** kubectl is a last resort.

**Why?**
- Our clients don't have kubectl — we must replicate their experience
- Every debugging gap is a product improvement opportunity
- If it can't be diagnosed via CLI, that's a bug in our observability

**The debugging ladder:**

| Priority | Tool | When |
|----------|------|------|
| 1st | `eve` CLI | **Always start here** |
| 2nd | `./bin/eh status` | Environment health |
| 3rd | `kubectl` | **Only when CLI is insufficient** |

**If you use kubectl to debug something:**
1. Note what information was missing from the CLI
2. Add it to `eve job diagnose`, `eve system logs`, or a new command
3. Update this skill with the new pattern

## Fast triage

- `eve system health`
- `eve job diagnose <id>`
- `eve job show <id> --verbose`
- `eve job follow <id>` or `eve job logs <id> --attempt N`

## Common blockers

- Phase not ready: `eve job update <id> --phase ready`
- Dependency blocks: `eve job dep list <id>`
- Environment gate: check diagnose output for env lock messages

## Provisioning failures (clone, hooks, workspace)

- Treat as fail-fast; fix root cause, do not skip steps.
- Check orchestrator and worker logs for the first failing step.
- For k8s runner failures, use `eve job runner-logs <id>`.

## Environment-specific logs

**CLI first (preferred):**
- `eve system logs api` — API service logs
- `eve system logs orchestrator` — Orchestrator logs
- `eve system logs worker` — Worker logs
- `eve job runner-logs <id>` — Runner pod logs for a specific job

**Local dev (files):**
- `/tmp/eve-api.log`, `/tmp/eve-orchestrator.log`, `/tmp/eve-worker.log`

**Docker (when CLI unavailable):**
- `docker logs eve-api -f`, `docker logs eve-orchestrator -f`

**kubectl (LAST RESORT — then improve the CLI):**
- `kubectl -n eve logs deployment/eve-orchestrator -f`
- If you need this, file an issue to add the capability to `eve system logs`

## Recursive skill distillation

- Add new failure modes and fixes to this skill as they appear.
- Split repeated triage patterns into new eve-dev skills if they grow large.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
