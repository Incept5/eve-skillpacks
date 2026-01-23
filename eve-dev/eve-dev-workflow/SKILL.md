---
name: eve-dev-workflow
description: Eve Horizon internal development workflow, runtime options, testing requirements, and pre-deployment guardrails. Use when making code, infra, or docs changes in the Eve Horizon repo.
---

# Eve Dev Workflow

Use this workflow for changes inside Eve Horizon.

## CRITICAL: First Step - Check Environment

**Before ANY build, test, or development activity:**

```bash
./bin/eh status
```

This tells you:
- What environments are running (k8s, docker, local)
- The correct `EVE_API_URL` to use
- Whether you own the k8s stack (see below)

## CRITICAL: K8s Ownership Permissions

Check `./bin/eh status` output for `K8s Owner: true/false`

### If `k8s_owner: true` (You Own the Stack)

You are **responsible** for the k8s stack. You CAN freely:
- Rebuild images (`./bin/eh k8s-image push`)
- Redeploy (`./bin/eh k8s deploy`)
- Restart services
- Run migrations
- Start/stop the cluster

**No approval needed** - this is your stack to manage.

### If `k8s_owner: false` (Another Instance Owns It)

You are a **guest** on someone else's stack. You:
- **CAN** run jobs and tests against the stack
- **CAN** use the CLI (`eve job create`, etc.)
- **CAN** fix code if stack errors occur
- **CANNOT** rebuild, redeploy, or restart k8s services
- **CANNOT** run `./bin/eh k8s deploy` or `./bin/eh k8s-image push`

If stack failures occur:
1. Fix the code that caused the issue
2. Ask the k8s owner to rebuild/redeploy
3. Or get explicit approval before touching k8s

To change ownership: `./bin/eh configure --k8s-owner` or `--no-k8s-owner`

## Modes (Mutually Exclusive)

| Mode | Command | API URL | When to Use |
|------|---------|---------|-------------|
| Local | `./bin/eh start local` | `http://localhost:4801` | Hot-reload development |
| Docker | `./bin/eh start docker` | `http://localhost:4801` | Integration tests |
| K8s | `./bin/eh k8s deploy` | `http://api.eve.lvh.me` | E2E tests |

**Local and Docker modes are exclusive** - starting one stops the other automatically.
K8s is separate (shared resource with ownership model).

Check `./bin/eh status` for current mode and exact URLs.

## Core Guardrails

- Treat CLI and API as the only interface; avoid direct DB edits in tests.
- Pre-deployment phase: simplify, refactor, delete dead code.
- Fail fast on provisioning errors; do not swallow errors or add workarounds.

## Starting Services

```bash
# Local mode - hot-reload development
./bin/eh start local          # DB container + local node processes
./bin/eh stop                 # Stop when done

# Docker mode - containerized stack
./bin/eh start docker         # All services in containers
./bin/eh stop                 # Stop when done

# K8s mode - e2e testing (requires k8s_owner)
./bin/eh k8s start && ./bin/eh k8s deploy
```

**Always use `./bin/eh start`** instead of raw pnpm/docker commands - it manages mode tracking and cleans up orphaned processes.

## Multi-Instance Support

Multiple repo checkouts can run integration tests in parallel:

```bash
# Configure unique instance
./bin/eh configure --instance eh2 --base-port 4900
```

Each instance gets:
- Isolated Docker Compose containers (via `COMPOSE_PROJECT_NAME`)
- Isolated ports (base_port + offsets)
- Isolated Postgres volumes

The k3d cluster is **shared** - only one instance should own it.

## Build and Test Verification

**Required for significant work:**

```bash
./bin/eh status              # Check environment first!
pnpm install                 # Refresh workspace links
pnpm build                   # Full build
pnpm test                    # Unit tests
./bin/eh test integration    # Integration tests
./bin/eh test e2e --env stack  # E2E (if k8s_owner)
```

## Done Criteria

- Tests pass for the changed scope
- Docs updated when behavior changes
- `AGENTS.md` update log appended for significant work

## Recursive Skill Distillation

- Capture repeated fixes, commands, and pitfalls as updates to this skill
- If a new workflow emerges, add a new eve-dev skill instead of bloating this one
- Update the eve-skillpacks README and ARCHITECTURE listings after changes
