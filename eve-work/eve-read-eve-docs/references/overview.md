# Eve Horizon Overview

## Load Eve Docs First

Before any work on or with Eve Horizon, load this skill and read the relevant references:

```bash
skill read eve-read-eve-docs
```

Then review the references that match your task. Start here for architecture and conventions; open `references/cli.md`, `references/manifest.md`, or `references/jobs.md` for specifics.

## Check Environment Status

Before ANY build, test, or development activity, run:

```bash
./bin/eh status
```

This shows what environments are running, the correct `EVE_API_URL` for each, port configuration, and multi-instance isolation status. Never assume URLs or ports.

### Environment Access

| Environment | API URL | When to Use |
|---|---|---|
| K8s (k3d) | `http://api.eve.lvh.me` | Manual tests, deployment testing |
| Docker Compose | `http://localhost:4801` | Integration tests, quick dev loop |
| Local pnpm dev | `http://localhost:4801` | Hot-reload development |

No port-forwarding required for K8s -- all services are accessible via Ingress.

## Developer Quick Start

```bash
./bin/eh status                                    # 0. Check what's running
./bin/eh k8s start && ./bin/eh k8s deploy          # 1. Start k3d cluster + deploy
export EVE_API_URL=http://api.eve.lvh.me           # 2. Set API URL
eve org ensure test-org --slug test-org             # 3. Use the CLI
./bin/eh k8s stop                                  # 4. Clean up
```

## Current State

**Phase**: Pre-MVP (K8s runtime + agent runtime + chat gateway complete).

**What exists**: monorepo with API, orchestrator, worker, agent runtime, and gateway services. Database with orgs, projects, environments, jobs, attempts, agents, teams, threads, integrations, schedules. Persistent environment deployment to K8s. Manifest variable interpolation. Ingress routing. Job execution with mclaude/zai harnesses. Agent/team/thread primitives with repo-first sync. Chat gateway with Slack integration. Agent runtime (org-scoped warm pods). CLI as npm package and local `./bin/eh` helpers. K8s local stack via k3d.

**What's next**: production domain configuration, registry-based image deployment, UI dashboards, Slack interactive approvals.

### Pre-Deployment Phase

No deployments to production. No real users. No backwards compatibility concerns. This means:

1. **Simplify aggressively** -- if something can be simpler, make it simpler.
2. **Refactor without fear** -- no migrations to maintain, no users to break.
3. **Delete ruthlessly** -- dead code, unused abstractions, speculative features.
4. **Question everything** -- every abstraction must earn its place.

## What Eve Horizon Is

Eve Horizon is a **job-first platform** that runs AI-powered skills against Git repos:

- **CLI-first**: humans and agents use the same CLI against the API.
- **Job-centric**: all work is tracked as jobs with phases, priorities, and dependencies.
- **Event-driven**: automation routes through an event spine in Postgres.
- **Skills-based**: reusable capabilities live as `SKILL.md` files in repos.
- **Isolated execution**: each job attempt runs in a fresh workspace with a cloned repo.
- **Staging-first**: default guidance targets staging; local dev is opt-in.

## Architecture Summary

```
User -> CLI -> API -> Orchestrator -> Worker -> Harness -> Agent
                |       |             |
                |       |             +-> Runner pods (k8s)
                |       +-> Postgres (jobs, attempts, events, secrets)
Chat -> Gateway +
Agent Runtime (warm pods) --+

ONLY URL needed: EVE_API_URL
CLI is a thin wrapper; all control flows through the API.
```

- **API**: single gateway for org/project/job CRUD, secrets, events, runs.
- **Orchestrator**: polls ready jobs, routes events to pipelines/workflows.
- **Worker**: clones repo, invokes harness, streams JSONL logs.
- **Harness**: wrapper for Claude/Codex/Gemini/Z.ai via `eve-agent-cli`.
- **Gateway**: routes inbound messages from Slack/Nostr to agents.
- **Agent Runtime**: org-scoped warm pods for low-latency chat responses.

**Key flows**:

1. Create job -> API validates -> stored in DB.
2. Orchestrator claims ready jobs -> creates JobWorkspace.
3. Worker invokes selected harness (mclaude/zai) -> streams JSONL logs.
4. Chat flow: Slack -> Gateway -> API routes -> jobs + threads -> Agent Runtime executes.
5. HITL review -> continue or complete.

## Key Decisions

| Decision | Rationale |
|---|---|
| mclaude + zai via cc-mirror | Claude Opus support and Z.ai via job-level selection |
| Job (not Task) terminology | Avoids collision with cc-mirror's Task tools |
| NestJS for backend | Clean architecture, TypeScript native |
| Hierarchical job IDs | `{slug}-{hash8}` root, `{parent}.{n}` children |
| Phase-based job lifecycle | idea -> backlog -> ready -> active -> review -> done |
| Single repo per project | Simplifies execution and config |
| CLI as thin REST wrapper | Single source of truth, no DB bypass |
| API as single gateway | CLI needs only `EVE_API_URL` |
| K8s runtime via k3d | Production-like local testing, runner pods for isolation |
| Agent runtime (warm pods) | Reduces per-message cold starts for chat workflows |
| Chat gateway + Slack mapping | Multi-tenant `team_id -> org_id` routing |
| Repo-first agents sync | Deterministic config via `--ref` |

## Conventions

- **Org/Project IDs**: `org_xxx`, `proj_xxx` (TypeID format).
- **Job IDs**: `{slug}-{hash8}` for root jobs (e.g., `myproj-a3f2dd12`).
- **Job phases**: `idea` -> `backlog` -> `ready` -> `active` -> `review` -> `done`/`cancelled`.
- **Priority**: 0-4 (P0=critical, P4=backlog, default=2).
- **K8s namespaces**: `eve-{orgSlug}-{projectSlug}-{envName}` for deployments.

### IDs and Formats

| Entity | Format | Example |
|---|---|---|
| Org ID | `org_...` | `org_abc123` |
| Project ID | `proj_...` | `proj_def456` |
| Job ID (root) | `{slug}-{hash8}` | `myproj-a3f2dd12` |
| Job ID (child) | `{parent}.{n}` | `myproj-a3f2dd12.1` |
| Build ID | `bld_...` | `bld_xyz789` |
| Release ID | `rel_...` | `rel_uvw012` |
| Managed DB Instance | `mdbi_...` | `mdbi_01abc` |
| Managed DB Tenant | `mdbt_...` | `mdbt_01def` |
| Event ID | `evt_...` | `evt_ghi345` |
| Pipeline Run ID | `prun_...` | `prun_jkl678` |
| Service Principal | `sp_...` | `sp_abc123` |
| SP Token | `spt_...` | `spt_def456` |
| Access Role | `role_...` | `role_ghi789` |
| Access Binding | `bind_...` | `bind_jkl012` |
| Attempt | UUID + `attempt_number` | (1, 2, 3...) |

## Development Workflow

### Environment Status First

Always start here:

```bash
./bin/eh status
export EVE_API_URL=http://api.eve.lvh.me     # for k8s
export EVE_API_URL=http://localhost:4801      # for docker/dev
```

### Build and Test

```bash
pnpm install          # Refresh workspace links
pnpm build            # Full build (all packages and apps)
pnpm test             # Unit tests (fast, no external deps)
./bin/eh test integration   # Integration tests (docker DB + local processes)
```

### Hot-Reload Development

```bash
./bin/eh start local    # DB container + local node processes (hot-reload)
./bin/eh start docker   # All services in containers
./bin/eh k8s start && ./bin/eh k8s deploy   # K8s stack (manual testing)
./bin/eh stop           # Stop current mode
```

Modes are exclusive -- starting a different mode stops the current one.

### Manual Testing (Observable Tests)

For k8s stack testing, use manual test scenarios with parallel job watching. Read `tests/manual/README.md` before running.

```bash
./bin/eh status                    # Must show k8s cluster "running"
eve system health --json           # Must return {"status":"ok"}
eve org ensure "manual-test-org" --slug manual-test-org --json
eve secrets import --org org_manualtestorg --file manual-tests.secrets
# Run scenarios from tests/manual/scenarios/
eve job follow <job-id>            # Stream logs in real-time
```

### CLI-First Debugging

Debug via the Eve CLI first. Always. Our clients do not have kubectl access -- replicate their experience. Every debugging gap is a product improvement opportunity.

**Debugging ladder**:

| Priority | Tool | When to Use |
|---|---|---|
| 1st | `eve` CLI | Always start here -- job status, logs, diagnose |
| 2nd | `./bin/eh status` | Environment health, connectivity |
| 3rd | `kubectl` | Only when CLI is insufficient -- then file an issue |

**CLI debugging commands**:

```bash
eve system health --json
eve job list --all --phase active
eve job show <id> --verbose
eve job follow <id>               # Stream harness logs
eve job logs <id>                 # Historical logs
eve job result <id>               # Exit status + outputs
eve job diagnose <id>             # Full diagnostic dump
eve env show <project> <env>      # Deployment health
```

### Build/Deploy Debugging Ladder

| Priority | Command | What It Shows |
|---|---|---|
| 1st | `eve pipeline logs <pipeline> <run-id> --follow` | Real-time streaming of all steps |
| 2nd | `eve pipeline logs <pipeline> <run-id>` | Snapshot with inline errors + hints |
| 3rd | `eve build diagnose <build_id>` | Full build state (last 30 lines of buildkit output) |
| 4th | `eve env diagnose <project> <env>` | K8s deployment diagnostics |
| 5th | `eve job diagnose <job_id>` | Full job execution details |

Pipeline build failures include the build ID in error output. Use `eve build diagnose <build_id>` to see the failed Dockerfile stage.

### Multi-Instance Support

Eve Horizon supports multiple repo checkouts running integration tests in parallel:

| Resource | Isolation | Shared |
|---|---|---|
| Docker Compose | Per-instance (via `EVE_INSTANCE` prefix) | -- |
| Postgres | Per-instance volume | -- |
| Ports | Per-instance (via `base_port`) | -- |
| k3d Cluster | -- | Shared (one `eve-local` cluster) |

Default ports: API=4801, Orchestrator=4802, DB=4803, Worker=4811. Config stored in `.eve-horizon.yaml` (gitignored). Check `./bin/eh status` for k8s ownership status before touching the shared cluster.

## Testing Architecture

Three-tier test pyramid:

| Tier | Type | What It Tests | Environment | Command |
|---|---|---|---|---|
| 1 | Unit | Pure logic, validators | None | `pnpm test` |
| 2 | Integration | API, job flows, secrets | Docker DB + local pnpm | `./bin/eh test integration` |
| 3 | Manual | Happy paths, real repos | K8s stack | See `tests/manual/` |

Integration tests use API endpoints, not direct DB queries. Test and dev use separate databases (`eve` for dev, `eve_test` for integration).

## Sister Repositories

| Repo | Expected Path | Purpose |
|---|---|---|
| eve-horizon-starter | `../eve-horizon-starter` | Starter template for new Eve projects |
| eve-horizon-fullstack-example | `../eve-horizon-fullstack-example` | Example fullstack app for deployment testing |
| eve-skillpacks | `../eve-skillpacks` | Published skill packs referenced by `skills.txt` |

Agents are free to make changes, commit, and push to `main` in these repos without explicit approval -- this is part of the normal development flow.

### Skillpacks Sync Obligation

When platform behavior changes in eve-horizon, the corresponding reference file in `eve-skillpacks/eve-work/eve-read-eve-docs/references/` must be updated. If `eve-read-eve-docs` is outdated, agents will make incorrect assumptions. See the `eve-docs-upkeep` skill for the full audit checklist.

## Build -> Release -> Deploy

The standard deployment flow:

1. **Build**: create container images from source code.
2. **Release**: capture a deployable snapshot (SHA + manifest + digests).
3. **Deploy**: apply release to an environment.

Pipelines orchestrate these steps as a job graph. See `references/builds-releases.md`.

## Event Spine

Events are stored in Postgres and routed by the orchestrator:

| Source | Description |
|---|---|
| `github` | Webhook events (push, pull_request) |
| `slack` | Chat messages and app mentions |
| `cron` | Scheduled triggers |
| `system` | Auto-emitted failure events (job.failed, pipeline.failed) |
| `runner` | Worker execution events (started, progress, completed, failed) |
| `manual` | User-created via CLI/API |
| `app` | Application-emitted |
| `chat` | Chat system events |

Triggers in the manifest map events to pipeline runs or workflow jobs. See `references/events.md`.

## Agents, Teams, and Chat

- **Agents**: defined in `agents.yaml`, synced via `eve agents sync`.
- **Teams**: defined in `teams.yaml`, coordinate multiple agents (fanout/council/relay).
- **Chat routing**: defined in `chat.yaml`, maps messages to agents via regex patterns.
- **Slack**: `@eve <agent-slug> <command>` routing.
- **Nostr**: relay-based subscription transport.

See `references/agents-teams.md`.

## Skills System

Reusable capabilities installed via `skills.txt` manifest:

- `SKILL.md` files with frontmatter metadata.
- Installed to `.agents/skills/` at clone time (legacy: `.agent/skills/`).
- Workers run `.eve/hooks/on-clone.sh` to install skills.
- Preferred flow: AgentPacks via `x-eve.packs` + `.eve/packs.lock.yaml`.

See `references/skills-system.md`.

## Key Rules

- **CLI only needs `EVE_API_URL`**; everything routes through the API.
- **Default to staging** for user guidance unless explicitly asked for local dev.
- **Pipelines and workflows are manifest-defined** and materialize into jobs.
- **Git controls live on jobs** (ref, branch, commit/push policies).
- **Agent slugs are org-unique** (enforced at sync time).
- **Secrets scopes stack**: project > user > org > system.
- **Service principals** provide machine identity for app backends (scoped JWT tokens).
- **Custom roles** are additive overlays on member/admin/owner (no deny rules).
- **Policy-as-code** via `.eve/access.yaml` enables reviewable, CI-friendly access config.
- **Fail fast on provisioning errors** -- swallow no errors, fix root causes.

## Reference Index

| Topic | Reference File |
|---|---|
| CLI commands | `references/cli.md` |
| Manifest schema | `references/manifest.md` |
| Events + triggers | `references/events.md` |
| Jobs | `references/jobs.md` |
| Builds + releases | `references/builds-releases.md` |
| Agents, teams, chat | `references/agents-teams.md` |
| Pipelines + workflows | `references/pipelines-workflows.md` |
| Secrets + auth | `references/secrets-auth.md` |
| Skills system | `references/skills-system.md` |
| Deploy + debug | `references/deploy-debug.md` |
| Harnesses | `references/harnesses.md` |
| Gateway plugins | `references/gateways.md` |

## Planned vs Current

System docs often separate **Current (Implemented)** from **Planned** sections. Treat "Planned" as non-functional unless the user confirms it is shipped.
