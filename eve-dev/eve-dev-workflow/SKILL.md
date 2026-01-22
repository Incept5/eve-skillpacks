---
name: eve-dev-workflow
description: Eve Horizon internal development workflow, runtime options, testing requirements, and pre-deployment guardrails. Use when making code, infra, or docs changes in the Eve Horizon repo.
---

# Eve Dev Workflow

Use this workflow for changes inside Eve Horizon.

## Core guardrails

- Treat CLI and API as the only interface; avoid direct DB edits in tests.
- Pre-deployment phase: simplify, refactor, delete dead code.
- Fail fast on provisioning errors; do not swallow errors or add workarounds.

## Local runtimes

- K8s (default): `./bin/eh k8s start` then `./bin/eh k8s deploy`
- Docker compose (quick loop): `./bin/eh docker start`
- Local watch: `./bin/eh test integration` (starts services with watch)
- Use `./bin/eh dev start` instead of `pnpm dev` to clean orphaned processes.

## Environment setup

- Set `EVE_API_URL=http://api.eve.lvh.me` for the local k8s stack.
- Use `EVE_API_URL=http://localhost:4701` for local dev or docker compose stacks.
- Use `./bin/eh configure` for instance config and k8s ownership.
- If `k8s_owner` is false, get confirmation before running k8s commands.

## Build and test verification (required for significant work)

- `pnpm install` (refresh workspace links)
- `pnpm build`
- `pnpm test`
- `./bin/eh test integration`
- Run `./bin/eh test e2e --env stack` for k8s or deployment changes.

## Done criteria

- Tests pass for the changed scope.
- Docs updated when behavior changes.
- `AGENTS.md` update log appended for significant work.

## Recursive skill distillation

- Capture repeated fixes, commands, and pitfalls as updates to this skill.
- If a new workflow emerges, add a new eve-dev skill instead of bloating this one.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
