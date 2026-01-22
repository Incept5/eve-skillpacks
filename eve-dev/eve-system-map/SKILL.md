---
name: eve-system-map
description: Navigate Eve Horizon architecture, docs, and code layout. Use when locating where to implement or debug platform behavior in the core repo.
---

# Eve System Map

Use this map to find the right docs and code paths quickly.

## Doc map

- System index: `docs/system/README.md`
- System overview: `docs/system/system-overview.md`
- Architecture: `ARCHITECTURE.md`, `docs/system/unified-architecture.md`
- Jobs: `docs/system/job-api.md`, `docs/system/job-cli.md`, `docs/system/job-control-signals.md`
- Manifest and deploy: `docs/system/manifest.md`, `docs/system/deployment.md`, `docs/system/secrets.md`
- Skills: `docs/system/skills.md`, `docs/system/skillpacks.md`
- Harnesses: `docs/system/agent-harness-design.md`, `docs/system/harness-execution.md`
- API contracts: `docs/system/openapi.yaml`

## Code map

- API service: `apps/api`
- Orchestrator: `apps/orchestrator`
- Worker and deployer: `apps/worker`
- CLI package: `packages/cli`
- Agent harness CLI: `packages/eve-agent-cli`
- Worker CLI: `packages/worker-cli`
- Shared libs: `packages/shared`, `packages/db`
- Dev tooling: `bin/eh`, `k8s/`, `docker/`

## Source of truth

- Treat `docs/system` as authoritative for current behavior.
- Use `AGENTS.md` for dev memory and conventions.
- Keep user-facing guidance in `README.md`.

## Recursive skill distillation

- Add new doc or code hotspots as they appear.
- Split large sections into a reference file if this skill grows.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
