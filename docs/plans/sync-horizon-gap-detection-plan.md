# Sync-Horizon Gap Detection Improvement Plan

> **Status**: Ready for implementation
> **Created**: 2026-02-28
> **Intent**: Add coverage auditing and cross-cutting skill triggers to sync-horizon so it catches structural gaps — not just incremental changes.

## The Problem

Sync-horizon is a change propagator. It answers: "what changed upstream?" and updates known targets. But it cannot answer: "are our docs complete?" or "did a new capability make an existing skill stale?"

This blind spot let 6 shipped primitives (KV store, memory namespaces, unified search, thread distillation, object store, FS sharing) accumulate across multiple syncs without the agent-memory skill ever being updated. The skill actively told agents these capabilities didn't exist.

## 5 Whys Summary

1. No `skill_triggers` entry maps anything to `eve-agent-memory` → it's invisible to sync
2. `eve-agent-memory` is cross-cutting (spans KV, memory, docs, threads, fs, search) → no single source doc triggers it
3. Sync-map only supports 1:1/N:1 source→target mappings → cross-cutting skills don't fit
4. No coverage audit phase → new CLI modules and system docs that aren't in the map are silently ignored
5. No staleness detection → skills making claims ("no KV store") are never validated against platform state

## Root Causes → Solutions

| Root Cause | Solution |
|---|---|
| No cross-cutting triggers | Add `composite_triggers` to sync-map |
| No coverage audit | Add Phase 1.5: Coverage Audit |
| No staleness detection | Add Phase 2.5: Staleness Check |
| Sync-map requires manual updates for new docs | Auto-detect unmapped source docs |

## Design

### Change 1: Extend `.sync-map.json` with `composite_triggers`

Current sync-map has `reference_docs` (source→reference) and `skill_triggers` (source→skills). Both are keyed by individual source paths.

Add a third section: `composite_triggers` — skills triggered when *any* of a set of capability-related sources change.

```json
{
  "reference_docs": { ... },
  "skill_triggers": { ... },
  "composite_triggers": {
    "eve-work/eve-agent-memory": {
      "description": "Cross-cutting skill spanning all storage and memory primitives",
      "watch_sources": [
        "packages/cli/src/commands/kv.ts",
        "packages/cli/src/commands/memory.ts",
        "packages/cli/src/commands/search.ts",
        "packages/cli/src/commands/thread.ts",
        "packages/cli/src/commands/fs.ts",
        "packages/cli/src/commands/docs.ts",
        "packages/cli/src/commands/db.ts",
        "packages/cli/src/commands/event.ts",
        "packages/cli/src/commands/job.ts",
        "docs/system/object-store-and-org-filesystem.md"
      ],
      "check_type": "any_changed"
    }
  }
}
```

When any `watch_sources` file appears in the `--stat` output, the composite skill gets a work item. The worker reads the diff for the changed source(s) and the skill's SKILL.md + references, then updates.

**Why this works**: Cross-cutting skills don't need every source to change — any one of them changing might introduce a new capability the skill should document. The worker is responsible for reading the skill's current content and deciding what needs updating.

### Change 2: Add Phase 1.5 — Coverage Audit

After getting the `--stat` output (Phase 1) and before planning work items (Phase 2), add a lightweight audit:

```
### Phase 1.5: Coverage Audit (orchestrator — lightweight)

Check for structural gaps: new source docs or CLI modules with no sync-map target.

1. List all docs/system/*.md files in eve-horizon:
   ```bash
   cd ../eve-horizon && ls docs/system/*.md
   ```

2. List all CLI command modules:
   ```bash
   cd ../eve-horizon && ls packages/cli/src/commands/*.ts
   ```

3. Cross-reference against `.sync-map.json`:
   - For each docs/system/*.md file, check if it appears in any `reference_docs` key
   - For CLI modules, check if the `packages/cli/src/commands/` glob covers them (it does by default, but new modules might warrant dedicated reference docs)

4. Report unmapped sources in the sync report under "Coverage Gaps":
   - New system docs with no reference mapping
   - Suggest whether each gap needs a new reference doc or an addition to an existing mapping

This phase does NOT read file contents — it only compares file lists against the map. Stays within the orchestrator's context budget.
```

**Why this works**: It catches the `object-store-and-org-filesystem.md` case — a new 785-line system doc appeared but was never mapped. The orchestrator flags it for human attention rather than silently ignoring it.

### Change 3: Add Phase 2.5 — Staleness Check (conditional)

When composite_triggers fire, add a staleness check work item for the affected skill:

```
### Worker Instructions: Staleness Check

> Read the target SKILL.md. Look for claims about what exists or doesn't exist:
> - "No dedicated KV store" / "no X available" / "not yet supported"
> - "Current Gaps" or "Workarounds" sections
> - Decision tables that might be missing rows for new primitives
>
> Cross-reference these claims against the current CLI commands:
>   cd ../eve-horizon && ls packages/cli/src/commands/*.ts
>
> Report: which claims are now stale? Which decision tables need new rows?
> Then update the skill accordingly.
```

**Why this works**: Instead of just propagating the diff, the worker validates the skill's own claims against current reality. This catches the "No dedicated KV store" problem directly.

### Change 4: Update the sync report template

Add two new sections to Phase 5 report:

```
### Coverage Gaps (unmapped sources)
- <list of docs/system/*.md files with no sync-map entry>
- <recommendation: create reference doc / add to existing mapping / ignore>

### Staleness Findings
- <skills that had stale claims, what was corrected>
```

## Implementation: Changes to SKILL.md

### Sync-map additions

Add to `.sync-map.json`:

```json
"composite_triggers": {
  "eve-work/eve-agent-memory": {
    "description": "Cross-cutting: all storage and memory primitives",
    "watch_sources": [
      "packages/cli/src/commands/kv.ts",
      "packages/cli/src/commands/memory.ts",
      "packages/cli/src/commands/search.ts",
      "packages/cli/src/commands/thread.ts",
      "packages/cli/src/commands/fs.ts",
      "packages/cli/src/commands/db.ts",
      "packages/cli/src/commands/event.ts",
      "packages/cli/src/commands/job.ts",
      "docs/system/object-store-and-org-filesystem.md"
    ],
    "check_type": "any_changed"
  }
}
```

### SKILL.md workflow additions

Insert Phase 1.5 after Phase 1 (Discover Changes):

> **Phase 1.5: Coverage Audit**
>
> List all `docs/system/*.md` files in eve-horizon. Cross-reference against `.sync-map.json` `reference_docs` keys. Report any unmapped docs under "Coverage Gaps" in the final report.
>
> Also check `composite_triggers`: for each entry, check if any of its `watch_sources` appear in the `--stat` output. If so, add a staleness-check work item for that skill.

Insert composite trigger handling into Phase 2 (Plan Work Items):

> For each `composite_triggers` entry where any `watch_sources` file appeared in `--stat`:
> - Create a work item: `Check and update <skill> for staleness`
> - Worker should read the skill's SKILL.md and references, check for stale claims against current platform state, and update accordingly
> - Include the specific changed source files so the worker knows what's new

Add to Phase 5 (Report):

> **Coverage Gaps** and **Staleness Findings** sections in the sync report template.

## Validation

After implementing:

1. Run a simulated sync from a commit before KV store shipped → verify composite_triggers fires for eve-agent-memory
2. Run coverage audit → verify object-store-and-org-filesystem.md is flagged as unmapped
3. Staleness check on eve-agent-memory → verify "No dedicated KV store" is caught and corrected
4. Normal syncs with no cross-cutting changes → verify composite_triggers doesn't fire spuriously

## Effort Estimate

- `.sync-map.json` update: add `composite_triggers` section — 5 minutes
- `SKILL.md` update: add Phase 1.5 + composite trigger handling + report sections — 30 minutes
- No new scripts needed — all checks are bash one-liners the orchestrator can run inline

## Future Considerations

- **Auto-suggest sync-map entries**: When coverage audit finds an unmapped doc > 100 lines, auto-suggest which reference_docs key to add it to (or suggest creating a new reference doc)
- **Periodic full audit**: Run the coverage audit even when there are no new commits, to catch drift from manual edits
- **Composite trigger granularity**: Consider `check_type: "n_or_more_changed"` for skills that should only trigger when multiple related sources change simultaneously (avoids noise from single-file typo fixes)
