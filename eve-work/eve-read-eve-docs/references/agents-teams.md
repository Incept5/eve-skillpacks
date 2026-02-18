# Agents, Teams + Chat Routing

## Use When
- You need to define or verify agent and team YAML for chat dispatch.
- You need to route inbound Slack/Nostr messages to specific personas or team workflows.
- You need to set or audit gateway discovery policy (`none`, `discoverable`, `routable`).

## Load Next
- `references/gateways.md` to map provider-specific message handling.
- `references/pipelines-workflows.md` for job-based escalation or workflow handoffs.
- `references/skills-system.md` when resolving agent pack prerequisites.

## Ask If Missing
- Confirm the target project/org manifest path and whether agents are repo-synced.
- Confirm whether routing is intended to be command-only or auto-routable in chat.
- Confirm if team dispatch should use fanout, relay, or lead/merge semantics.

## Overview

Agents, teams, and chat routes are repo-first YAML configurations synced to Eve via `eve agents sync`. The repo is the source of truth. Agents are personas with skills and policies; teams are dispatch groups that coordinate agents; routes map inbound chat messages to targets.

## agents.yaml

Define agents with capabilities, access, policies, and gateway visibility.

```yaml
version: 1
agents:
  mission-control:
    slug: mission-control           # org-unique, ^[a-z0-9][a-z0-9-]*$
    description: "Primary orchestration agent"
    skill: eve-orchestration        # required
    workflow: nightly-audit          # optional
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
      policy: routable               # none | discoverable | routable
      clients: [slack]               # omit = all providers
    schedule:
      heartbeat_cron: "*/15 * * * *"
```

### Agent Slug Rules

- Lowercase alphanumeric + dashes, org-unique, enforced at sync.
- Used for Slack routing: `@eve <slug> <command>`.
- Set a default: `eve org update org_xxx --default-agent <slug>`.

### Gateway Discovery Policy

Control which agents are visible and directly addressable from external chat gateways. Internal dispatch (teams, pipelines, routes) is always unaffected.

| Policy | `@eve agents list` | `@eve <slug> msg` | Internal dispatch |
|--------|--------------------|--------------------|-------------------|
| `none` | Hidden | Rejected | Works |
| `discoverable` | Visible | Rejected (hint) | Works |
| `routable` | Visible | Works | Works |

Resolution order: pack `gateway.default_policy` (base, defaults to `none`) -> agent `gateway.policy` -> project overlay.

## teams.yaml

Define teams with a lead agent, members, and a dispatch mode.

```yaml
version: 1
teams:
  review-council:
    lead: mission-control
    members: [code-reviewer, security-auditor]
    dispatch:
      mode: fanout
      max_parallel: 3
      lead_timeout: 300
      member_timeout: 300
      merge_strategy: majority

  ops:
    lead: ops-lead
    members: [deploy-agent, monitor-agent]
    dispatch:
      mode: relay
```

### Team Dispatch Modes

| Mode | Behavior |
|------|----------|
| `fanout` | Root job + parallel child jobs per member |
| `council` | All agents respond, results merged by strategy |
| `relay` | Sequential delegation from lead through members |

## chat.yaml

Define routing rules with explicit target prefixes.

```yaml
version: 1
default_route: route_default
routes:
  - id: deploy-route
    match: "deploy|release|ship"
    target: agent:deploy-agent
    permissions:
      project_roles: [admin, member]

  - id: review-route
    match: "review|PR|pull request"
    target: team:review-council

  - id: route_default
    match: ".*"
    target: team:ops
    permissions:
      project_roles: [member, admin, owner]
```

### Route Matching

- `match` is a regex tested against message text.
- First match wins; fallback to `default_route` if none match.
- Target prefixes: `agent:<key>`, `team:<key>`, `workflow:<name>`, `pipeline:<name>`.

## Syncing Configuration

```bash
# Sync from committed ref (production)
eve agents sync --project proj_xxx --ref abc123def456...

# Sync local state (development)
eve agents sync --project proj_xxx --local --allow-dirty

# Preview effective config without syncing
eve agents config --repo-dir ./my-app
```

Sync resolves AgentPacks from `x-eve.packs`, deep-merges pack agents/teams/chat with local overrides, validates org-wide slug uniqueness, and pushes to the API.

### Pack Overlay

Local YAML overlays pack defaults via deep merge. Use `_remove: true` to drop a pack agent.

```yaml
agents:
  pack-agent:
    harness_profile: my-custom-profile   # override pack default
  unwanted-agent:
    _remove: true                         # remove from pack
```

## Warm Pods / Agent Runtime

Warm pods are pre-provisioned org-scoped containers that eliminate cold starts for chat-triggered jobs. Routing is org-sticky.

```bash
eve agents runtime-status --org org_xxx
```

Output: pod name, status (ready/degraded/unhealthy), capacity, last heartbeat.

Data model: `agent_runtime_pods` (heartbeat + capacity), `agent_placements` (pod selection), `agent_state` (status + heartbeat).

## Coordination Threads

When teams dispatch work, a coordination thread links the parent job to all child agents.

- Thread key: `coord:job:{parent_job_id}`
- Child agents receive `EVE_PARENT_JOB_ID` environment variable
- Derive the thread key: `coord:job:${EVE_PARENT_JOB_ID}`
- End-of-attempt summaries auto-post to the coordination thread
- Coordination inbox: `.eve/coordination-inbox.md` (regenerated from recent messages at job start)

### Coordination Message Kinds

| Kind | Purpose |
|------|---------|
| `status` | Automatic end-of-attempt summary |
| `directive` | Lead-to-member instruction |
| `question` | Member-to-lead question |
| `update` | Progress update from a member |

### Thread CLI

```bash
eve thread messages <thread-id> --since 5m      # list recent messages
eve thread post <thread-id> --body '{"kind":"update","body":"Phase 1 complete"}'
eve thread follow <thread-id>                    # stream in real-time
```

## Supervision

Monitor a job tree and coordinate team execution.

```bash
eve supervise                       # supervise current job
eve supervise <job-id> --timeout 60 # supervise specific job with timeout
```

Long-polls child job events for the lead agent.

## Slack Integration

### Routing Commands (in Slack)

```
@eve <agent-slug> <command>         # direct to specific agent
@eve agents list                    # list available agent slugs
@eve agents listen <agent-slug>     # subscribe agent to channel or thread
@eve agents unlisten <agent-slug>   # unsubscribe agent
@eve agents listening               # list active listeners
```

### Thread-Level vs Channel-Level Listeners

- Issue `listen` in a channel: creates a **channel-level** listener (all messages in the channel).
- Issue `listen` inside a thread: creates a **thread-level** listener (only messages in that thread).
- Multiple agents can listen to the same channel or thread.
- Listening uses `message.channels` events; explicit `@eve` commands use `app_mention`.

### Slack CLI Setup

```bash
eve integrations slack connect --org org_xxx --team-id T123 --token xoxb-test
eve integrations list --org org_xxx
eve integrations test <integration_id> --org org_xxx
```

### Default Agent

```bash
eve org update org_xxx --default-agent mission-control
```

When a message does not start with a known slug, Eve routes to the org default agent with the full message as the command.

## Chat Simulation

Test the full routing pipeline without a live provider.

```bash
eve chat simulate --project <id> --team-id T123 --channel-id C123 --user-id U123 --text "hello" --json
```

Returns `thread_id` and `job_ids` showing how the message would be dispatched.

## API Endpoints

```
POST /projects/{id}/agents/sync       # sync agents/teams/chat config
POST /projects/{id}/agents/config     # get effective merged config
GET  /agents                           # list agents
GET  /teams                            # list teams

GET  /threads/{id}/messages            # list thread messages
POST /threads/{id}/messages            # post to thread
GET  /threads/{id}/follow              # stream thread messages (SSE)

POST /chat/route                       # route inbound message
POST /chat/simulate                    # simulate chat message
POST /chat/listen                      # subscribe agent to channel/thread
POST /chat/unlisten                    # unsubscribe agent
GET  /chat/listeners                   # list active listeners
```
