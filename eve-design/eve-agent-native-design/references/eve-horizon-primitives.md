# Eve Horizon Platform Primitives

Distillation of the platform primitives available for agent-native app building.

## Jobs

The fundamental unit of work. Create, orchestrate, and compose into hierarchies.

- `eve job create` — Create a job with optional parent and dependencies
- `eve job dep add` — Wire dependency edges between jobs
- Depth propagation: child jobs inherit and increment depth from parents
- Parallel decomposition: break work into concurrent sub-jobs
- Control signals: `eve job signal` for pause/resume/cancel
- Completion: `json-result` with `eve.status` for explicit signals

## Agents & Teams

Define personas, dispatch modes, and chat routing.

- `agents.yaml` — Declare agent personas with skills, model, system prompt
- `teams.yaml` — Group agents into teams with routing rules
- Chat gateway routes messages to agents by slug
- Dispatch modes: direct, round-robin, skill-based routing
- Service principals for machine-to-machine agent access

## Pipelines & Workflows

Deterministic sequences and on-demand functions.

- Pipelines: ordered stages with gates and approvals
- Workflows: reusable functions triggered by events or manual invocation
- Both defined in `.eve/manifest.yaml`
- Compose existing CLI commands — no special pipeline runtime

## Gateway

Multi-provider chat with agent discovery.

- Routes messages to agents by slug
- Supports multiple chat providers
- Gateway discovery policy controls agent visibility
- New agents are instantly addressable once declared

## Builds & Releases

Immutable artifacts with promotion workflows.

- `eve build create` + `eve build run` — Build container images
- Digest-based artifacts ensure immutability
- Release references `build_id` for exact artifact tracking
- Environment promotion: staging -> production
- Image tagging: `:local` for dev, git SHA or semver for pipelines

## Skills

Knowledge distribution and progressive disclosure.

- `skills.txt` — Declare skill sources per project
- `eve-worker skills install` — Install from repos or local paths
- Skills installed to `.agent/skills/` on clone
- Progressive disclosure: index skills route to detailed skills
- Skill packs group related skills by domain

## Events

Automation triggers and event spine.

- Event-driven pipeline triggers
- Webhook integrations
- Event spine connects platform components
- Custom event types for app-specific automation

## Managed Resources

Platform-provided infrastructure (emerging).

- DBaaS: Managed Postgres provisioning
- Container registry: Native image storage with `registry.host` in manifest
- Secret management: `eve secret set/get` with environment scoping
- Future: multi-registry, air-gapped scenarios, lifecycle management

## Environment & Context

Every agent environment includes:

- `EVE_API_URL` — Platform API endpoint
- `EVE_PUBLIC_API_URL` — Public-facing API endpoint
- `EVE_PROJECT_ID` — Current project identifier
- `EVE_ORG_ID` — Organization identifier
- `EVE_ENV_NAME` — Current environment (staging, production)
- Skills loaded from `.agent/skills/` provide domain vocabulary
