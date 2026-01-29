# Eve Horizon Overview (Current)

## What Eve Horizon Is

Eve Horizon is a **job-first platform** that runs AI-powered skills against Git repos. It is:
- **CLI-first**: humans and agents use the same CLI against the API.
- **Job-centric**: all work is tracked as jobs with phases, priorities, and dependencies.
- **Event-driven**: automation routes through an event spine in Postgres.
- **Skills-based**: reusable capabilities live as OpenSkills `SKILL.md` files in repos.
- **Isolated execution**: each job attempt runs in a fresh workspace with a cloned repo.
- **Staging-first**: default guidance targets staging; local dev is opt-in.

## Core Architecture

```
User/Agent -> CLI -> API -> Orchestrator -> Worker -> Harness -> Agent
                  |       |             |
                  |       |             +-> Runner pods (k8s)
                  |       +-> Postgres (jobs, attempts, events, secrets)
```

- **API**: single gateway for org/project/job CRUD, secrets, events, runs.
- **Orchestrator**: polls ready jobs, routes events to pipelines/workflows.
- **Worker**: clones repo, invokes harness, streams JSONL logs.
- **Harness**: wrapper for Claude/Codex/Gemini/Z.ai via `eve-agent-cli`.

## IDs and Formats

- **Org ID**: `org_...` (TypeID)
- **Project ID**: `proj_...` (TypeID)
- **Job ID (root)**: `{slug}-{hash8}` (e.g., `myproj-a3f2dd12`)
- **Job ID (child)**: `{parent}.{n}` (e.g., `myproj-a3f2dd12.1`)
- **Attempts**: UUID + `attempt_number` (1,2,3...).

## Job Lifecycle

Phases: `idea` -> `backlog` -> `ready` -> `active` -> `review` -> `done` (or `cancelled`).
- Default phase on create: `ready` (schedulable).
- Priorities: 0-4 (P0 highest).

## Event Spine (Current)

Events are stored in Postgres and routed by the orchestrator. Sources include GitHub, Slack, cron, system, and manual. Triggers in the manifest map events to pipeline runs or workflow jobs.

## Key Rules (Current)

- **CLI only needs `EVE_API_URL`**; everything routes through the API.
- **Default to staging** for user guidance unless explicitly asked for local dev.
- **Pipelines and workflows are manifest-defined** and materialize into jobs.
- **Git controls live on jobs** (ref, branch, commit/push policies).

## Planned vs Current

System docs often separate **Current (Implemented)** from **Planned** sections. Treat "Planned" as non-functional unless the user confirms it is shipped.
