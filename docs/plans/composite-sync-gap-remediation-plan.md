# Composite Sync Gap Remediation Plan

> **Status**: Ready for implementation
> **Created**: 2026-02-28
> **Scope**: Feb 14 – Feb 28, 2026 (790a9e1..7ecf171, ~80 commits)
> **Intent**: Fix every documentation gap that composite triggers should have caught in the last 2 weeks. Update cross-cutting skills, create missing reference docs, and wire the sync infrastructure so this class of gap doesn't recur.

---

## The Problem

Sync-horizon propagates incremental changes through 1:1 mappings (`reference_docs`, `skill_triggers`). This works for known source→target pairs. It cannot catch:

1. **New system docs** with no mapping — silently ignored
2. **Cross-cutting skills** that synthesize from many sources — no single source triggers them
3. **Stale claims** in skills — a skill asserting "this doesn't exist" is never re-checked

The `.sync-map.json` now has a `composite_triggers` section (one entry for `eve-agent-memory`), but it was never wired into the sync workflow. Over 2 weeks, this let 6 shipped primitives, 3 new system docs (~1,400 lines), and multiple cross-cutting skill impacts go unaddressed.

**Worst case**: `eve-agent-memory/SKILL.md` actively tells agents that KV store, unified search, and memory namespaces *don't exist*. All three shipped Feb 16.

## Guiding Principles

1. **State-today fidelity** — only document shipped behavior. Remove stale "gaps" that are now shipped.
2. **Progressive disclosure** — SKILL.md teaches patterns and routes; references hold CLI details.
3. **One source of truth per primitive** — each primitive has exactly one reference section. Others cross-reference.
4. **Agent-native thinking** — teach agents *when* to use each primitive, not just *how*.

---

## 2-Week Audit: What Shipped vs What We Caught

### Feature Area 1: Agent Memory Platform (Feb 16)

Commit `ddb136d` — 6 new primitives:

| Primitive | CLI Module | Caught by Sync? | Status in Skills |
|-----------|-----------|-----------------|------------------|
| Agent KV Store | `kv.ts` (new) | No | **Missing** — SKILL.md says "No dedicated KV store" |
| Agent Memory Namespaces | `memory.ts` (new) | No | **Missing** — SKILL.md says "No automatic context carryover" |
| Unified Search | `search.ts` (new) | No | **Missing** — SKILL.md says "No vector search" |
| Thread Distillation | `thread.ts` updated | No | **Missing** |
| Context Auto-Hydration | worker changes | No | **Missing** |
| Document Lifecycle | API changes | No | **Missing** |

### Feature Area 2: Object Store & Org Filesystem (Feb 20-24)

Commits `e6df653`→`332486c` — new unified storage layer:

| Change | Source | Caught? | Status |
|--------|--------|---------|--------|
| 785-line system doc | `object-store-and-org-filesystem.md` (new) | **No mapping** | No reference doc exists |
| Presigned URL transport | same doc | No | Not in any skill |
| App object store (`x-eve.object_store`) | same doc + `manifest.md` | Partially (manifest ref) | Missing from manifest SKILL.md |
| Share tokens + public paths | `fs.ts` changes | Yes (CLI glob) | In CLI ref only |
| Async text indexing | same doc | No | Not in any skill |

### Feature Area 3: Slack Integration (Feb 22-26)

Commits `f15ec51`→`556304e` — 8-phase Slack gap closure:

| Change | Source | Caught? | Status |
|--------|--------|---------|--------|
| Integrations doc expanded (234 lines) | `integrations.md` | **No mapping** | No reference doc |
| Gateway simulate | `chat-gateway.md` | Yes (gateways.md) | Updated |
| Identity resolution (3-tier) | `integrations.md` | No | Not in any skill |
| Membership requests, OAuth flow | `integrations.md` | No | Not in any skill |
| Block Kit formatting | `chat-gateway.md` | Yes (gateways.md) | Updated |
| CLI: `identity link`, `integrations` | `identity.ts`, `integrations.ts` | Yes (CLI glob) | In CLI ref |

### Feature Area 4: App SSO & Auth SDK (Feb 26-28)

Commits `5ad9167`→`7ecf171`:

| Change | Source | Caught? | Status |
|--------|--------|---------|--------|
| `app-sso-integration.md` (185 lines, new) | system doc | No mapping | Partially covered in `eve-auth-and-secrets`, no dedicated reference doc |
| `eve-auth-sdk.md` (373 lines, new) | system doc | **No mapping** | **No reference doc** |
| `@eve-horizon/auth`, `@eve-horizon/auth-react` | package code | Partially (skill_triggers) | In eve-auth-and-secrets ✅ |

### Feature Areas 5-6: GitHub Integration + CLI & UX

GitHub `eve github setup` — caught by CLI glob. All CLI/UX changes (admin users, --version, chat simulate, ollama pull/models, fs share/publish) — caught by CLI glob. No composite gap.

---

## Gap Summary

### Critical — Actively Wrong

| Gap | Skill | What's Wrong |
|-----|-------|-------------|
| 6 memory primitives undocumented | `eve-agent-memory` | SKILL.md claims KV, search, context carryover don't exist — all shipped |
| CLI command names wrong | `eve-agent-memory` | Docs examples use deprecated `create/get`; `update` is not a real subcommand. Canonical examples should use `write/read` |

### High — Missing Reference Docs

| Source Doc | Lines | Target | Exists? |
|-----------|-------|--------|---------|
| `object-store-and-org-filesystem.md` | 785 | `references/object-store-filesystem.md` | No |
| `eve-auth-sdk.md` | 373 | `references/auth-sdk.md` | No |
| `integrations.md` + `app-sso-integration.md` | 419 | `references/integrations.md` | No |

### Medium — Cross-Cutting Skill Gaps

| Skill | What's Missing |
|-------|---------------|
| `eve-agent-memory/references/storage-primitives.md` | KV, Memory, Search, Thread Distill sections; comparison matrix rows |
| `eve-read-eve-docs/SKILL.md` Task Router | No routes for object store, auth SDK, integrations intents |
| `eve-manifest-authoring/SKILL.md` | No `x-eve.object_store` declaration pattern |
| `eve-fullstack-app-design/SKILL.md` | No SSO or object store app design patterns |

---

## Implementation Plan

### Phase 1: Fix eve-agent-memory SKILL.md

**Goal**: Remove false claims, add 6 new primitives, update patterns and decision table.

#### 1a. Add new primitives to Storage Primitives by Timescale

**Short-Term** — add:
- **Agent KV Store** — lightweight operational state with optional TTL. Use for: feature flags, rate counters, agent state machines, deduplication keys. Namespace-partitioned (`eve kv set --namespace <ns>`).

**Long-Term** — add:
- **Agent Memory Namespaces** — curated knowledge stored as org docs with agent-scoped path conventions. Categories: `learnings`, `decisions`, `runbooks`, `context`, `conventions`. Supports confidence scores and tags. Use for: accumulated expertise, decision logs, operational runbooks.

**Shared** — add:
- **Unified Search** — single query across memory, docs, threads, attachments, events. Use for: finding relevant prior knowledge before starting work.
  - Note: docs search has text/semantic/hybrid options where surfaced; keep the cross-source unified-search entry explicit about current scope and behavior.
- **Thread Distillation** — convert thread conversations into memory docs (`eve thread distill`). Use for: preserving valuable discussion outcomes.

#### 1b. Fix CLI command names

- `eve docs create` → `eve docs write` (legacy alias; canonical `write`)
- `eve docs get` → `eve docs read`
- `eve docs update` → `eve docs write` (no dedicated `update` subcommand; write updates existing docs)

#### 1c. Add new patterns

**Pattern 5: Search-Before-Write** — search existing knowledge before creating new docs.

```
eve search --org $ORG --query "auth retry patterns" --sources memory,docs →
  If relevant result exists: read and update it
  If no result: create new memory doc
```

**Pattern 6: KV State Machine** — use KV store to track multi-step agent workflows.

```
eve kv set --agent reviewer --namespace workflow --key "pr-123" --value '{"phase":"review","started":"..."}' --ttl 86400
```

**Pattern 7: Thread-to-Knowledge Pipeline** — distill valuable thread conversations into searchable memory.

```
eve thread distill $THREAD_ID --agent reviewer --category learnings --key "auth-discussion-findings"
```

#### 1d. Replace "Current Gaps and Workarounds"

Remove false claims:
- "No dedicated KV store" → shipped
- "No vector search" → unified search shipped
- "No automatic context carryover" → agent memory namespaces cover this

Replace with actual current gaps:
- No automatic context carryover at job start (startup sequence is manual)
- No automatic thread-to-knowledge distillation (manual `eve thread distill` exists, but no cron/trigger)
- Vector/semantic cross-source search not yet documented as end-to-end shipped behavior (current cross-source `eve search` remains text-oriented)

#### 1e. Update decision table

| Question | Answer → Primitive |
|---|---|
| Need it only during this job? | Workspace files |
| Need lightweight state with TTL? | Agent KV Store |
| Need to pass data to parent/children? | Job attachments |
| Need rolling conversation context? | Threads |
| Need to distill a conversation into knowledge? | Thread distillation → Agent Memory |
| Need curated, categorized agent knowledge? | Agent Memory Namespaces |
| Need versioned, searchable documents? | Org Document Store |
| Need file-tree sync with local editing? | Org Filesystem |
| Need app-scoped binary/object storage? | Object Store (manifest) |
| Need structured queries (SQL)? | Managed database |
| Need to encode a reusable workflow? | Skills |
| Need reactive notifications? | Event spine |
| Need to search across everything? | Unified Search |

### Phase 2: Update storage-primitives.md reference

Add three new sections:

**Section 12: Agent KV Store**
- Scope: Org (agent-scoped)
- CLI: `eve kv set/get/list/mget/delete`
- Options: `--namespace`, `--ttl`, `--agent`
- Database table: `agent_kv` (id, org_id, agent_slug, namespace, key, value JSONB, ttl_seconds, expires_at)

**Section 13: Agent Memory Namespaces**
- Scope: Org (agent-scoped or shared)
- CLI: `eve memory set/get/list/delete`
- Options: `--agent`, `--shared`, `--category`, `--key`, `--confidence`, `--tags`
- Namespace convention: `/agents/{slug}/memory/{category}/{key}.md` or `/agents/shared/memory/...`
- Categories: learnings, decisions, runbooks, context, conventions

**Section 14: Unified Search**
- Scope: Org
- CLI: `eve search --org --query --sources --limit --agent`
- Sources: memory, docs, threads, attachments, events
- Response: source, snippet, score, timestamps, locators

Update existing sections:
- **Section 3 (Threads)**: add `eve thread distill` command
- **Section 6 (Org Filesystem)**: add share/publish commands, share tokens, public paths
- **Comparison Matrix**: add KV, Memory, Search rows

### Phase 3: Create missing reference docs

#### 3a. `eve-work/eve-read-eve-docs/references/object-store-filesystem.md`

Distill from: `../eve-horizon/docs/system/object-store-and-org-filesystem.md` (785 lines)

Structure:
1. **Overview** — unified storage layer (MinIO local, S3/GCS/R2/Tigris in cloud)
2. **Org Filesystem** — presigned URL transport, sync protocol, events, conflict resolution
3. **Text Indexing** — async indexing of synced text files into org_documents for full-text search
4. **App Object Stores** — manifest `x-eve.object_store` declaration, auto-injected env vars (STORAGE_ENDPOINT, STORAGE_ACCESS_KEY, etc.)
5. **Share Tokens** — time-limited revocable URLs for file sharing
6. **Public Paths** — permanent unauthenticated access to path prefixes
7. **Access Control** — orgfs scope in access groups, path ACLs
8. **CLI Quick Reference** — table of all fs commands

#### 3b. `eve-work/eve-read-eve-docs/references/auth-sdk.md`

Distill from: `../eve-horizon/docs/system/eve-auth-sdk.md` (373 lines)

Structure:
1. **Overview** — two packages: `@eve-horizon/auth` (backend) and `@eve-horizon/auth-react` (frontend)
2. **Backend: `@eve-horizon/auth`** — exports (`eveUserAuth`, `eveAuthGuard`, `eveAuthConfig`, `verifyEveToken`), token types, org membership checking
3. **Frontend: `@eve-horizon/auth-react`** — `EveAuthProvider`, `EveLoginGate`, `useEveAuth` hook, session lifecycle
4. **Token Claims** — EveTokenClaims interface, user vs job vs service_principal tokens
5. **Migration Guide** — from manual JWKS to SDK middleware
6. **Auto-Injected Env Vars** — `EVE_SSO_URL`, `EVE_ORG_ID`, `EVE_API_URL`

#### 3c. `eve-work/eve-read-eve-docs/references/integrations.md`

Distill from: `../eve-horizon/docs/system/integrations.md` (234 lines) + `app-sso-integration.md` (185 lines)

Structure:
1. **Overview** — integrations connect external providers (Slack, GitHub) to Eve events and chat routing
2. **Slack Integration** — connect, webhook endpoints, routing (`@eve <slug>`), auth (signing secret, bot token)
3. **GitHub Integration** — `eve github setup`, webhook auto-configuration
4. **Identity Resolution** — 3 tiers: email auto-match, CLI self-service link, admin approval
5. **Membership Requests** — pending/approved/denied workflow, admin approval via Slack/CLI
6. **CLI Quick Reference** — `eve integrations`, `eve identity link`, `eve github setup`

### Phase 4: Update cross-cutting skills

#### 4a. `eve-se/eve-manifest-authoring/SKILL.md`

Add `x-eve.object_store` declaration pattern:
```yaml
services:
  api:
    x-eve:
      object_store:
        buckets:
          - name: uploads
            public: false
```

Document auto-injected env vars: `STORAGE_ENDPOINT`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`, `STORAGE_BUCKET`, `STORAGE_FORCE_PATH_STYLE`.

#### 4b. `eve-design/eve-fullstack-app-design/SKILL.md`

- Add SSO integration as a standard app design pattern (reference `@eve-horizon/auth` + `@eve-horizon/auth-react`)
- Add object store usage for app assets/uploads
- Cross-reference `references/auth-sdk.md` and `references/object-store-filesystem.md`

### Phase 5: Wire sync infrastructure

#### 5a. Update `.sync-map.json` `reference_docs`

Add:
```json
"docs/system/object-store-and-org-filesystem.md": "eve-work/eve-read-eve-docs/references/object-store-filesystem.md",
"docs/system/eve-auth-sdk.md": "eve-work/eve-read-eve-docs/references/auth-sdk.md",
"docs/system/integrations.md + docs/system/app-sso-integration.md": "eve-work/eve-read-eve-docs/references/integrations.md"
```

#### 5b. Update `.sync-map.json` `composite_triggers`

Replace the current broad entry with specific watch sources, and add new entries:

```json
"composite_triggers": {
  "eve-work/eve-agent-memory": {
    "description": "Cross-cutting: all storage and memory primitives",
    "watch_sources": [
      "docs/system/",
      "packages/cli/src/commands/"
    ]
  },
  "eve-se/eve-auth-and-secrets": {
    "description": "Cross-cutting: auth, identity, SSO, and credential patterns",
    "watch_sources": [
      "docs/system/auth.md",
      "docs/system/secrets.md",
      "docs/system/identity-providers.md",
      "docs/system/app-sso-integration.md",
      "docs/system/eve-auth-sdk.md",
      "docs/system/agent-secret-isolation.md",
      "packages/cli/src/commands/auth.ts",
      "packages/cli/src/commands/identity.ts",
      "packages/cli/src/commands/secrets.ts"
    ]
  },
  "eve-se/eve-manifest-authoring": {
    "description": "Cross-cutting: manifest shape changes including new x-eve extensions",
    "watch_sources": [
      "docs/system/manifest.md",
      "docs/system/object-store-and-org-filesystem.md",
      "packages/cli/src/commands/manifest.ts"
    ]
  },
  "eve-design/eve-fullstack-app-design": {
    "description": "Cross-cutting: app design patterns including auth, storage, and integrations",
    "watch_sources": [
      "docs/system/app-sso-integration.md",
      "docs/system/eve-auth-sdk.md",
      "docs/system/object-store-and-org-filesystem.md",
      "docs/system/integrations.md",
      "docs/system/manifest.md"
    ]
  }
}
```

#### 5c. Update `eve-work/eve-read-eve-docs/SKILL.md` Task Router

Add routing entries:
- Object storage, file sharing, app buckets, presigned URLs → `references/object-store-filesystem.md`
- Auth SDK, `@eve-horizon/auth`, `@eve-horizon/auth-react`, app SSO middleware → `references/auth-sdk.md`
- Integrations, Slack connect, GitHub setup, identity linking → `references/integrations.md`

---

## Worker Dispatch Plan

8 workers, 3 waves:

### Wave 1 (parallel — no dependencies)

| Worker | Target File | Source Material |
|--------|------------|-----------------|
| W1 | `eve-work/eve-agent-memory/SKILL.md` | Phase 1 spec above + `../eve-horizon/packages/cli/src/commands/{kv,memory,search,thread}.ts` |
| W2 | `eve-work/eve-agent-memory/references/storage-primitives.md` | Phase 2 spec above + same CLI sources |
| W3 | `eve-work/eve-read-eve-docs/references/object-store-filesystem.md` (create) | `../eve-horizon/docs/system/object-store-and-org-filesystem.md` |
| W4 | `eve-work/eve-read-eve-docs/references/auth-sdk.md` (create) | `../eve-horizon/docs/system/eve-auth-sdk.md` |
| W5 | `eve-work/eve-read-eve-docs/references/integrations.md` (create) | `../eve-horizon/docs/system/integrations.md` + `app-sso-integration.md` |

### Wave 2 (parallel — after Wave 1)

| Worker | Target File | Depends On |
|--------|------------|------------|
| W6 | `eve-se/eve-manifest-authoring/SKILL.md` | W3 (object store ref exists) |
| W7 | `eve-design/eve-fullstack-app-design/SKILL.md` | W4, W5 (auth-sdk + integrations refs exist) |

### Wave 3 (sequential — after all content workers)

| Worker | Target Files | Depends On |
|--------|-------------|------------|
| W8 | `.sync-map.json` + `eve-read-eve-docs/SKILL.md` Task Router | W3-W5 (final file paths confirmed) |

---

## Validation Checklist

After all workers complete:

### Content accuracy
- [ ] `eve-agent-memory/SKILL.md` mentions all 13+ storage primitives
- [ ] No section claims shipped features are missing (KV, search, memory namespaces, thread distill)
- [ ] CLI commands match actual `eve` CLI (`write` not `create`, `read` not `get`)
- [ ] `storage-primitives.md` comparison matrix has rows for KV, Memory, Search

### Reference docs exist and are routed
- [ ] `references/object-store-filesystem.md` exists — covers presigned URLs, app buckets, share tokens, text indexing
- [ ] `references/auth-sdk.md` exists — covers `@eve-horizon/auth`, `@eve-horizon/auth-react`, token types, migration
- [ ] `references/integrations.md` exists — covers Slack, GitHub, identity resolution tiers
- [ ] All three are routed in `eve-read-eve-docs/SKILL.md` Task Router

### Sync infrastructure
- [ ] `.sync-map.json` `reference_docs` has entries for object-store, auth-sdk, integrations
- [ ] `.sync-map.json` `composite_triggers` has 4 entries (agent-memory, auth, manifest, fullstack-app)
- [ ] `watch_paths` covers all new source docs

### Cross-cutting skills
- [ ] `eve-manifest-authoring/SKILL.md` documents `x-eve.object_store` with env var injection
- [ ] `eve-fullstack-app-design/SKILL.md` documents SSO and object store app patterns

### Compliance
- [ ] State-today scan passes (no planned/roadmap content in any touched file)
- [ ] Progressive access intact (SKILL.md routes, references hold detail)

---

## Out of Scope

- Wiring composite triggers into the sync-horizon SKILL.md workflow (see `sync-horizon-gap-detection-plan.md`)
- Implementing automatic context carryover (platform feature, not docs)
- Vector/semantic cross-source search documentation (beyond current text-oriented cross-source implementation)
- Creating an `eve-se/eve-integrations` skill (potential future — identity resolution + provider management could warrant a dedicated skill)
