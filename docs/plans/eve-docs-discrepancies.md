# Eve Docs Discrepancies: Skill References vs Eden Reference Project

*Discovered 25 March 2026 while building Appendix D of the probate engine product vision.*

This documents gaps, misleading patterns, and outright errors in the `/eve-read-eve-docs` skill references when compared against the **Eden** reference project (`../eve-horizon/eden`), which is the canonical Eve Horizon application.

---

## 1. Manifest `name:` field undocumented

**Docs say:** Top-level fields are `schema`, `project`, `registry`, `services`, `environments`, `pipelines`, `workflows`, `versioning`, `x-eve`. The minimal example uses `project: my-app`. (`manifest.md:156–166`)

**Eden does:** Uses `name: eden` and `description: "AI-first requirements platform..."` at top level. No `project:` field at all.

**Impact:** An agent following the docs will write `project: probate-engine` instead of `name: probate-engine`. It's unclear whether both are valid, or only `name:` works in practice.

**Ref:** `references/manifest.md` lines 154–166

---

## 2. `x-eve.permissions` on services is undocumented

**Docs say:** The Eve Service Extensions table lists `role`, `ingress`, `api_spec`, `api_specs`, `cli`, `external`, `connection_url`, `worker_type`, `files`, `storage`, `managed`, `object_store`. No mention of `permissions`. (`manifest.md:225–241`)

**Eden does:** The API service has `x-eve.permissions: [jobs:write, events:write, threads:write]`. This grants the deployed service the ability to create jobs and emit events — essential for any app that triggers workflows.

**Impact:** Without this, an app deployed on Eve cannot invoke workflows or emit events via the API. An agent building a new Eve app would omit this and wonder why `POST /events` returns 403.

**Ref:** `references/manifest.md` lines 225–241

---

## 3. `with_apis` format inconsistency between docs

**Docs say (manifest.md):** Workflow `with_apis` is typed as `string[]` — shown as bare strings: `with_apis: [coordinator]` (`manifest.md:654–655, 675`)

**Docs say (agents-teams.md):** Agent `with_apis` uses full objects: `{service: api, description: ...}` (`agents-teams.md:48–50`)

**Eden does:** Uses full object format **everywhere** — both in agent definitions and in workflow definitions:
```yaml
with_apis:
  - service: api
    description: Eden Story Map API for reading map state and creating changesets
```

**Impact:** An agent following `manifest.md` will write `with_apis: [api]` in workflows. If the platform requires the object format, this silently fails or produces incorrect API injection.

**Ref:** `references/manifest.md` lines 654–655, 675; `references/pipelines-workflows.md` lines 213–214, 253–268

---

## 4. Workflow `condition` on steps is undocumented

**Docs say:** Workflow step fields are `name`, `depends_on`, `agent.name`, `with_apis`. No mention of `condition`. (`manifest.md:670–675`, `pipelines-workflows.md:229–234`)

**Eden does:** Uses conditional step execution:
```yaml
steps:
  - name: triage
    agent:
      name: question-triage
  - name: evolve
    depends_on: [triage]
    condition: "triage.status == 'needs_changes'"
    agent:
      name: question-agent
```

**Impact:** This is a significant orchestration feature — conditional branching in workflow DAGs. An agent designing workflows cannot discover this capability from the docs.

**Ref:** `references/manifest.md` lines 670–675, `references/pipelines-workflows.md` (no mention)

---

## 5. Agent `context` block undocumented

**Docs say:** Agent schema covers `slug`, `alias`, `description`, `skill`, `workflow`, `harness_profile`, `access`, `policies`, `with_apis`, `gateway`, `schedule`. No mention of `context`. (`agents-teams.md:26–56`)

**Eden does:** Several agents declare a `context` block for memory and coordination thread access:
```yaml
synthesis:
  context:
    memory:
      agent: shared
      categories: [decisions, conventions, context]
      max_items: 20
    threads:
      coordination: true
      max_messages: 30
```

**Impact:** Agents that need shared memory or access to coordination thread history cannot discover how to declare this. This is important for agents that need to maintain state across invocations or read team coordination context.

**Ref:** `references/agents-teams.md` lines 26–56

---

## 6. Agent `name` field undocumented

**Docs say:** The agent example schema shows `slug`, `description`, etc. but no `name` field for a human-readable display name. (`agents-teams.md:26–56`)

**Eden does:** Every agent has a `name:` field:
```yaml
pm-coordinator:
  slug: pm
  name: "PM Coordinator"
  description: "PM coordinator — triages requests..."
```

**Impact:** Minor — the `slug` is documented. But `name` appears to be the human-readable label shown in `@eve agents list` and team dispatch UI. Without it, agents get unnamed entries.

**Ref:** `references/agents-teams.md` lines 26–56

---

## 7. Eve SDK quick-start recommends `eveUserAuth()`, not `eveAuth()`

**Docs say (eve-sdk.md):** Quick-start example uses `eveUserAuth()`:
```typescript
app.use(eveUserAuth());  // Parse tokens (non-blocking)
```
(`eve-sdk.md:40–47`)

**Docs say (auth-sdk.md):** Deep reference recommends `eveAuth()` as the unified middleware for apps serving both users and agents, calling it "Recommended for New Apps". (`auth-sdk.md:92–133`)

**Eden does:** Uses `eveAuth()`:
```typescript
app.use(eveAuth());
```

**Impact:** An agent building a new Eve app reads the quick-start first (as designed) and uses `eveUserAuth()`. This middleware silently ignores agent job tokens, meaning all agent API calls fail with no user context. The unified `eveAuth()` should be the quick-start recommendation since nearly every Eve app serves both users and agents.

**Ref:** `references/eve-sdk.md` lines 40–47; `references/auth-sdk.md` lines 92–133

---

## 8. Pack `imports` schema not documented

**Docs say:** Packs are mentioned in `manifest.md` under `x-eve.packs` with source/ref resolution, lock files, and overlay customization. The `eve/pack.yaml` file is mentioned once as "full AgentPacks (`eve/pack.yaml`)" but its schema is never shown. (`manifest.md:780–837`)

**Eden does:** Has a full pack descriptor:
```yaml
# eve/pack.yaml
version: 1
id: eden
imports:
  agents: eve/agents.yaml
  teams: eve/teams.yaml
  workflows: eve/workflows.yaml
  chat: eve/chat.yaml
  x_eve: eve/x-eve.yaml
gateway:
  default_policy: none
```

**Impact:** An agent building a distributable pack cannot discover the `imports` schema, `id` field, `gateway.default_policy`, or which YAML files can be imported. This is the structural glue that connects the `eve/` directory to the pack system.

**Ref:** `references/manifest.md` lines 780–837

---

## 9. `eve/x-eve.yaml` for harness profiles not shown

**Docs say:** Harness profiles are defined in the manifest under `x-eve.agents.profiles` (`manifest.md:758–776`). No mention of defining them in a separate `eve/x-eve.yaml` file.

**Eden does:** Defines profiles in `eve/x-eve.yaml`:
```yaml
version: 1
agents:
  profiles:
    coordinator:
      - harness: claude
        model: sonnet
        reasoning_effort: medium
    expert:
      - harness: claude
        model: sonnet
        reasoning_effort: high
```

This file is imported by `eve/pack.yaml` via `x_eve: eve/x-eve.yaml`.

**Impact:** An agent following the docs puts profiles directly in the manifest. This works but doesn't integrate with the pack system — profiles won't be distributed with the pack.

**Ref:** `references/manifest.md` lines 758–776

---

## 10. `eve/workflows.yaml` as a pack-distributed file not documented

**Docs say:** "Workflows are defined in the manifest and invoked as jobs." (`pipelines-workflows.md:192–193`). All examples show workflows inside `.eve/manifest.yaml`.

**Eden does:** Defines workflows in **both** `.eve/manifest.yaml` (operational) AND `eve/workflows.yaml` (pack-distributed). The pack system imports from `eve/workflows.yaml`.

**Impact:** An agent building workflows only puts them in the manifest. When the project is packaged as an AgentPack for distribution, the workflows are not included — they're trapped in the manifest which isn't part of the pack import system.

**Ref:** `references/pipelines-workflows.md` lines 192–193, `references/manifest.md` lines 632–682

---

## 11. Missing `ingress.alias` in service extensions

**Docs say:** Ingress is documented as `{ public: true|false, port: number }`. (`manifest.md:230`)

**Eden does:** Uses `alias: eden` on ingress:
```yaml
x-eve:
  ingress:
    public: true
    port: 80
    alias: eden
```

**Impact:** Without `alias`, the app gets a verbose auto-generated hostname. The alias creates a clean vanity URL. An agent building a user-facing app would miss this and end up with `web.orgslug-projectslug-sandbox.eh1.incept5.dev` instead of `eden.eh1.incept5.dev`.

**Ref:** `references/manifest.md` line 230

---

## 12. Self-referencing internal URL pattern not documented

**Docs say:** Platform injects `EVE_API_URL` (Eve platform API), `EVE_PUBLIC_API_URL`, etc. (`manifest.md:506–533`)

**Eden does:** Also sets a custom self-referencing URL for service-to-service calls within the same project:
```yaml
environment:
  EDEN_API_URL: "http://${ENV_NAME}-api:3000"
```

**Impact:** This pattern is essential for apps that emit events back to their own API or need internal health checks. Without it, an app either hard-codes URLs or uses the Eve platform API URL (wrong target).

**Ref:** `references/manifest.md` lines 506–533

---

## Summary

| # | Issue | Severity | Type | Status |
|---|---|---|---|---|
| 1 | `name:` top-level field undocumented | Medium | Missing | Fixed — `manifest.md` |
| 2 | `x-eve.permissions` on services undocumented | High | Missing | Fixed — `8d943cf` |
| 3 | `with_apis` format inconsistent across docs | High | Contradictory | Fixed — `manifest.md`, `pipelines-workflows.md` |
| 4 | Workflow step `condition` undocumented | Medium | Missing | Fixed — `848a191` |
| 5 | Agent `context` block undocumented | Medium | Missing | Fixed — `agents-teams.md` |
| 6 | Agent `name` field undocumented | Low | Missing | Fixed — `agents-teams.md` |
| 7 | SDK quick-start recommends wrong middleware | High | Misleading | Fixed — `eve-sdk.md` |
| 8 | Pack `imports` schema not documented | High | Missing | Fixed — `manifest.md` |
| 9 | `eve/x-eve.yaml` for profiles not shown | Medium | Missing | Fixed — `manifest.md` |
| 10 | `eve/workflows.yaml` pack distribution not documented | Medium | Missing | Fixed — `pipelines-workflows.md` |
| 11 | `ingress.alias` not in service extensions | Low | Missing | Fixed — `manifest.md` |
| 12 | Self-referencing internal URL pattern not documented | Low | Missing | Fixed — `manifest.md` |

All 12 issues resolved as of 25 March 2026.

**High severity** = will cause an agent to produce broken or non-functional config.
**Medium severity** = agent will miss a useful capability or produce suboptimal config.
**Low severity** = cosmetic or convenience feature missed.
