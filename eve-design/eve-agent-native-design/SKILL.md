---
name: eve-agent-native-design
description: Design agent-native applications on Eve Horizon. Apply parity, granularity, composability, and emergent capability principles to make apps that agents can build, operate, and extend naturally.
triggers:
  - agent native
  - agent native design
  - design for agents
  - agent first
  - agentic app
  - agent native app
---

# Agent-Native Design for Eve Horizon

Build applications where agents are first-class citizens — not afterthoughts.

## When to Use

Load this skill when:
- Designing a new application or API on Eve
- Evaluating whether an existing app is agent-friendly
- Adding features that agents should be able to use
- Deciding between putting logic in code vs. in prompts

## The Four Principles

### 1. Parity — Agents Can Do Everything Users Can

Every user action must have an agent-equivalent path.

**On Eve**: The CLI IS the parity layer. If a user can do it through `eve ...`, an agent can too. When building your app, apply the same principle:

| Check | How |
|-------|-----|
| Can agents create/read/update/delete every entity? | Map UI actions to CLI/API equivalents |
| Are there UI-only workflows? | Expose them as API endpoints or CLI commands |
| Can agents discover what's available? | Provide `list` operations for every entity type |

**Eve pattern**: `eve job create`, `eve job list`, `eve job show`, `eve job update` — full CRUD via CLI.

### 2. Granularity — Atomic Primitives, Not Bundled Logic

Features emerge from agent loops, not monolithic tools.

**Wrong**: `deploy_and_monitor(app)` — bundles judgment into code
**Right**: `eve build create`, `eve build run`, `eve env deploy`, `eve job follow` — agent decides the sequence

**On Eve**: The manifest defines WHAT (services, pipelines), the agent decides HOW and WHEN to compose them.

**Design test**: To change behavior, do you edit prose (prompts/skills) or refactor code? If code — your tools aren't atomic enough.

### 3. Composability — New Features = New Prompts

When tools are atomic and parity exists, you add capabilities by writing prompts, not code.

**Eve example**: The `eve-pipelines-workflows` skill adds pipeline composition capability. No new CLI commands needed — the skill teaches agents to compose existing `eve pipeline` and `eve workflow` commands.

**Your app**: If adding a feature requires new API endpoints, you may be bundling logic. Consider whether existing primitives can be composed differently.

### 4. Emergent Capability — Agents Surprise You

Build atomic tools. Agents compose unexpected solutions. You observe patterns. Optimize common patterns. Repeat.

**Eve example**: Agents compose `eve job create --parent` + `eve job dep add` + depth propagation to build arbitrary work hierarchies. The platform didn't prescribe this — agents discovered it from atomic primitives.

## Eve-Specific Design Patterns

### Files as Universal Interface
- Agents know `cat`, `grep`, `mv`, `mkdir`
- Use `.eve/manifest.yaml` as single source of truth — agents read and edit it
- Agent configs live in repo files (`agents.yaml`, `teams.yaml`) — not hidden in databases

### Context Injection
- System prompts include available resources and capabilities
- Skills provide domain vocabulary (what a "job" is, what a "pipeline" does)
- Eve injects `EVE_API_URL`, `EVE_PROJECT_ID`, `EVE_ORG_ID`, `EVE_ENV_NAME` into every environment

### Explicit Completion Signals
- Jobs return `json-result` with `eve.status` ("success", "failed", "waiting")
- No heuristic completion detection — explicit signals always
- Track progress at task level with phase transitions

### Dynamic Capability Discovery
- `eve job list` discovers available work
- `eve agents list` discovers available agents
- Skills system auto-discovers capabilities at install time
- Gateway routes messages to agents by slug — new agents are instantly addressable

## The Success Checklist

Before shipping, verify:

- [ ] Every UI action has a CLI/API equivalent (parity)
- [ ] Tools do one thing; agent decides composition (granularity)
- [ ] Adding capability = adding a skill/prompt, not code (composability)
- [ ] Agent can handle requests you didn't explicitly design for (emergent)
- [ ] Manifest and config files are the source of truth (files as interface)
- [ ] Environment provides context injection (EVE_* vars, skills)
- [ ] Completion is explicit, not heuristic (json-result with eve.status)

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Agent as router only | Let agents act, not just route |
| Workflow-shaped tools (`analyze_and_deploy`) | Break into atomic primitives |
| UI-only actions | Maintain parity — add CLI/API paths |
| Context starvation | Inject resources via skills and env vars |
| Gates without reason | Default to open; keep primitives available |
| Heuristic completion | Use explicit completion signals |
| Static API mapping | Use dynamic capability discovery |

## Reference

See `references/eve-horizon-primitives.md` for the full platform primitives catalog.

For the source philosophy: `../eve-horizon/docs/ideas/agent-native-design.md`
For platform primitives analysis: `../eve-horizon/docs/ideas/platform-primitives-for-agentic-apps.md`
