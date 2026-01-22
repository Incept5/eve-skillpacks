---
name: eve-platform-debugging
description: Debug Eve Horizon platform failures and job execution issues across API, orchestrator, worker, and runner. Use when jobs are stuck, failing, or system health is unclear in the core platform.
---

# Eve Platform Debugging

Start CLI-first, then drill into service logs.

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

- Local dev: `/tmp/eve-api.log`, `/tmp/eve-orchestrator.log`, `/tmp/eve-worker.log`
- Docker: `docker logs eve-api -f`, `docker logs eve-orchestrator -f`, `docker logs eve-worker -f`
- K8s: `eve system logs api`, `eve system logs orchestrator`, `eve system logs worker`
- Fallback k8s: `kubectl -n eve logs deployment/eve-orchestrator -f`

## Recursive skill distillation

- Add new failure modes and fixes to this skill as they appear.
- Split repeated triage patterns into new eve-dev skills if they grow large.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
