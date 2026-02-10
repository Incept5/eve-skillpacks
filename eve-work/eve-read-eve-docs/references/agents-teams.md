# Agents, Teams + Chat Routing (Current)

## Overview

Agents, teams, and chat routes are configured via YAML files in the project repo and synced to Eve via `eve agents sync`. This is a **repo-first** model — the repo is the source of truth.

## agents.yaml

Defines individual agents with their capabilities, access, and policies.

```yaml
version: 1
agents:
  mission-control:
    name: "Mission Control"
    slug: mission-control           # org-unique, lowercase alphanumeric + dashes
    description: "Primary orchestration agent"
    role: orchestrator
    skill: eve-orchestration        # required — skill name
    workflow: nightly-audit          # optional — trigger workflow
    harness_profile: primary-orchestrator
    access:
      envs: [staging, production]
      services: [api, web]
      api_specs: [openapi]
    policies:
      permission_policy: auto_edit   # auto_edit | never | yolo
      git:
        commit: auto                 # never | manual | auto | required
        push: on_success             # never | on_success | required
    gateway:
      policy: routable             # visible + directly addressable via chat
      clients: [slack]             # optional: restrict to specific providers
    schedule:
      heartbeat_cron: "*/15 * * * *" # optional recurring trigger

  code-reviewer:
    name: "Code Reviewer"
    slug: code-reviewer
    description: "Reviews PRs"
    skill: eve-code-review
    harness_profile: primary-reviewer
    # no gateway block → inherits pack default (none if pack sets it, or none for standalone)
```

### Agent Slug Rules

- Format: `^[a-z0-9][a-z0-9-]*$` (lowercase alphanumeric + dashes)
- Scope: **org-unique** (enforced at sync time)
- Used for Slack routing: `@eve <slug> <command>`
- Set a default: `eve org update org_xxx --default-agent mission-control`

### Agent Fields Reference

| Field | Required | Description |
|-------|----------|-------------|
| `name` | No | Display name |
| `slug` | No | Org-unique identifier (auto-generated from key if omitted) |
| `description` | No | Human-readable description |
| `role` | No | Agent role label |
| `skill` | **Yes** | Skill name (must be installed) |
| `workflow` | No | Workflow to invoke |
| `harness_profile` | No | Profile from `x-eve.agents.profiles` |
| `access.envs` | No | Allowed environments |
| `access.services` | No | Allowed services |
| `access.api_specs` | No | Allowed API spec types |
| `policies.permission_policy` | No | `auto_edit` / `never` / `yolo` |
| `policies.git.commit` | No | `never` / `manual` / `auto` / `required` |
| `policies.git.push` | No | `never` / `on_success` / `required` |
| `gateway.policy` | No | `none` / `discoverable` / `routable` (see Gateway Discovery below) |
| `gateway.clients` | No | Restrict to specific providers (e.g., `[slack]`). Omit = all |
| `schedule.heartbeat_cron` | No | Cron expression for periodic triggers |

## Gateway Discovery Policy

Controls which agents are visible and routable from external chat gateways (Slack, Nostr). Internal dispatch (teams, pipelines, chat.yaml routes) is **always unaffected**.

### Policy Values

| Policy | `@eve agents list` | `@eve <slug> msg` | Team / Pipeline dispatch |
|--------|--------------------|--------------------|--------------------------|
| `none` | Hidden | Rejected | Always works |
| `discoverable` | Visible | Rejected (with hint) | Always works |
| `routable` | Visible | Works | Always works |

### Resolution Order

```
Pack gateway.default_policy          (base — defaults to none)
  → Agent gateway.policy override    (pack author intent)
    → Project overlay                (project owner final say)
```

### Pack-Level Default

Set in `eve/pack.yaml`:

```yaml
gateway:
  default_policy: none     # all agents hidden unless they opt in
```

Standalone agents (no pack) with no `gateway` block default to `none`.

### Examples

```yaml
# In pack agents.yaml — chatbot opts in
agents:
  intake:
    gateway: { policy: routable }      # users can talk to this
  reviewer:
    # no gateway block → inherits pack default (none)
```

```yaml
# Project overlay — expose a pack agent that was hidden
agents:
  reviewer:
    gateway: { policy: routable }      # project decides to expose
```

## teams.yaml

Defines teams of agents with coordination modes.

```yaml
version: 1
teams:
  review-council:
    lead: mission-control            # agent key (required)
    members: [code-reviewer, security-auditor]
    dispatch:
      mode: fanout                   # fanout | council | relay
      max_parallel: 3
      lead_timeout: 300              # ms
      member_timeout: 300            # ms
      merge_strategy: majority       # optional

  ops:
    lead: ops-lead
    members: [deploy-agent, monitor-agent]
    dispatch:
      mode: relay                    # sequential delegation
```

### Team Dispatch Modes

| Mode | Behavior |
|------|----------|
| `fanout` | Root job + child jobs per member (parallel execution) |
| `council` | All agents respond with analysis, results merged |
| `relay` | Sequential delegation from lead to members |

### Team Fields Reference

| Field | Required | Description |
|-------|----------|-------------|
| `lead` | **Yes** | Agent key for team lead |
| `members` | No | Array of agent keys |
| `dispatch.mode` | No | `fanout` / `council` / `relay` |
| `dispatch.max_parallel` | No | Max parallel executions (integer) |
| `dispatch.lead_timeout` | No | Lead timeout in ms |
| `dispatch.member_timeout` | No | Member timeout in ms |
| `dispatch.merge_strategy` | No | How to merge results (e.g., `majority`) |

## chat.yaml

Defines chat routing rules for inbound messages.

```yaml
version: 1
default_route: mission-control       # fallback agent
routes:
  - id: deploy-route
    match: "deploy|release|ship"     # regex pattern
    target: deploy-agent             # agent slug, or "agent:key" / "team:key"
    permissions:
      project_roles: [admin, member]
      envs: [staging]

  - id: review-route
    match: "review|PR|pull request"
    target: "team:review-council"    # target a team
```

### Route Matching

- `match` is a regex pattern tested against message text
- First match wins
- If no route matches, `default_route` is used
- Target can be: bare agent slug, `agent:key`, or `team:key`

## Syncing to Eve

```bash
# Sync from committed ref (production)
eve agents sync --project proj_xxx --ref abc123def456...

# Sync local state (development)
eve agents sync --project proj_xxx --local --allow-dirty

# Preview without syncing
eve agents config --repo-dir ./my-app
```

The sync command:
1. Resolves AgentPacks from `x-eve.packs`
2. Merges pack agents/teams/chat with local overrides
3. Validates slug uniqueness (org-wide)
4. Pushes configuration to the API

### Pack Overlay System

When using AgentPacks, local YAML files overlay pack defaults:

- **Deep merge**: Local fields merge into pack fields
- **`_remove`**: Remove specific items from pack config

```yaml
# Local agents.yaml overlay
version: 1
agents:
  pack-agent:
    harness_profile: my-custom-profile  # override pack default
  unwanted-agent:
    _remove: true                        # remove from pack
```

## Agent Runtime

Agents can run on **warm pods** — pre-provisioned containers per org that reduce cold-start time for chat workflows.

```bash
# Check agent runtime status
eve agents runtime-status --org org_xxx
```

Output shows: pod name, status (ready/degraded/unhealthy), capacity, last heartbeat.

## Coordination Threads

When teams dispatch work, coordination happens via threads:

- Thread key: `coord:job:{parent_job_id}`
- Child agents receive `EVE_PARENT_JOB_ID` environment variable
- End-of-attempt summaries auto-post to coordination thread
- Coordination inbox: `.eve/coordination-inbox.md` (regenerated at job start)

```bash
# View coordination messages
eve thread messages <thread-id> --since 5m

# Post to thread
eve thread post <thread-id> --body '{"kind":"update","body":"Phase 1 complete"}'

# Follow thread in real-time
eve thread follow <thread-id>
```

## Supervision

```bash
eve supervise [<job-id>] [--timeout 60]
```

Monitors job tree and coordinates team execution.

## Slack Integration

### Routing Commands (in Slack)

```
@eve <agent-slug> <command>        → Direct to specific agent
@eve agents list                   → List available agents
@eve agents listen <agent-slug>    → Subscribe agent to channel
@eve agents unlisten <agent-slug>  → Unsubscribe agent
@eve agents listening              → List active listeners
```

### Setting Up Slack

```bash
eve integrations slack connect --org org_xxx --team-id T123 --token xoxb-test
eve integrations list --org org_xxx
eve integrations test <integration_id> --org org_xxx
```

### Default Agent

Set an org-wide default agent for unmatched messages:

```bash
eve org update org_xxx --default-agent mission-control
```

## API Endpoints

```
POST /projects/{project_id}/agents/sync    # Sync agents/teams/chat
POST /projects/{project_id}/agents/config  # Get effective config
GET  /agents                                # List agents
GET  /teams                                 # List teams

GET  /threads/{id}/messages                 # List thread messages
POST /threads/{id}/messages                 # Post to thread
GET  /threads/{id}/follow                   # Stream thread messages

POST /chat/route                            # Route inbound message
POST /chat/simulate                         # Simulate chat message
POST /chat/listen                           # Subscribe agent to channel
POST /chat/unlisten                         # Unsubscribe agent
GET  /chat/listeners                        # List active listeners
```
