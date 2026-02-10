# Eve Horizon Overview (Current)

## What Eve Horizon Is

Eve Horizon is a **job-first platform** that runs AI-powered skills against Git repos. It is:
- **CLI-first**: humans and agents use the same CLI against the API.
- **Job-centric**: all work is tracked as jobs with phases, priorities, and dependencies.
- **Event-driven**: automation routes through an event spine in Postgres.
- **Skills-based**: reusable capabilities live as `SKILL.md` files in repos.
- **Isolated execution**: each job attempt runs in a fresh workspace with a cloned repo.
- **Staging-first**: default guidance targets staging; local dev is opt-in.

## Core Architecture

```
User/Agent -> CLI -> API -> Orchestrator -> Worker -> Harness -> Agent
                  |       |             |
                  |       |             +-> Runner pods (k8s)
                  |       +-> Postgres (jobs, attempts, events, secrets)
Chat -> Gateway --+
Agent Runtime (warm pods) --+
```

- **API**: single gateway for org/project/job CRUD, secrets, events, runs.
- **Orchestrator**: polls ready jobs, routes events to pipelines/workflows.
- **Worker**: clones repo, invokes harness, streams JSONL logs.
- **Harness**: wrapper for Claude/Codex/Gemini/Z.ai via `eve-agent-cli`.
- **Gateway**: routes inbound messages from Slack/Nostr to agents.
- **Agent Runtime**: org-scoped warm pods for low-latency chat responses.

## IDs and Formats

| Entity | Format | Example |
|--------|--------|---------|
| Org ID | `org_...` | `org_abc123` |
| Project ID | `proj_...` | `proj_def456` |
| Job ID (root) | `{slug}-{hash8}` | `myproj-a3f2dd12` |
| Job ID (child) | `{parent}.{n}` | `myproj-a3f2dd12.1` |
| Build ID | `bld_...` | `bld_xyz789` |
| Release ID | `rel_...` | `rel_uvw012` |
| Event ID | `evt_...` | `evt_ghi345` |
| Pipeline Run ID | `prun_...` | `prun_jkl678` |
| Attempt | UUID + `attempt_number` | (1, 2, 3...) |

## Job Lifecycle

Phases: `idea` -> `backlog` -> `ready` -> `active` -> `review` -> `done` (or `cancelled`).
- Default phase on create: `ready` (schedulable).
- Priorities: 0-4 (P0 highest).

## Build → Release → Deploy

The standard deployment flow:

1. **Build**: Create container images from source code
2. **Release**: Capture a deployable snapshot (SHA + manifest + digests)
3. **Deploy**: Apply release to an environment

Pipelines orchestrate these steps as a job graph. See `references/builds-releases.md`.

## Event Spine (Current)

Events are stored in Postgres and routed by the orchestrator. Sources include:

| Source | Description |
|--------|-------------|
| `github` | Webhook events (push, pull_request) |
| `slack` | Chat messages and app mentions |
| `cron` | Scheduled triggers |
| `system` | Auto-emitted failure events (job.failed, pipeline.failed) |
| `runner` | Worker execution events (started, progress, completed, failed) |
| `manual` | User-created via CLI/API |
| `app` | Application-emitted |
| `chat` | Chat system events |

Triggers in the manifest map events to pipeline runs or workflow jobs. See `references/events.md` for the complete catalog.

## Agents, Teams + Chat

- **Agents**: defined in `agents.yaml`, synced to API via `eve agents sync`
- **Teams**: defined in `teams.yaml`, coordinate multiple agents (fanout/council/relay)
- **Chat routing**: defined in `chat.yaml`, maps messages to agents via regex patterns
- **Slack integration**: `@eve <agent-slug> <command>` routing
- **Nostr integration**: relay-based subscription transport

See `references/agents-teams.md` for schemas and configuration.

## Skills System

Reusable capabilities installed via `skills.txt` manifest:
- `SKILL.md` files with frontmatter metadata
- Installed to `.agent/skills/` at clone time
- Workers run `.eve/hooks/on-clone.sh` to install skills

See `references/skills-system.md`.

## Key Rules (Current)

- **CLI only needs `EVE_API_URL`**; everything routes through the API.
- **Default to staging** for user guidance unless explicitly asked for local dev.
- **Pipelines and workflows are manifest-defined** and materialize into jobs.
- **Git controls live on jobs** (ref, branch, commit/push policies).
- **Agent slugs are org-unique** (enforced at sync time).
- **Secrets scopes stack**: project > user > org > system.

## Reference Index

| Topic | Reference File |
|-------|---------------|
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
