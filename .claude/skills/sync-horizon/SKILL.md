---
name: sync-horizon
description: Sync eve-skillpacks with latest eve-horizon changes. Reads git log, identifies affected skills, updates reference docs and skills, tracks sync point.
---

# Sync Horizon

Synchronize eve-skillpacks with the latest state of the eve-horizon repository.

## Prerequisites

- The eve-horizon repo must be at `../eve-horizon` (sibling directory)
- `.sync-state.json` must exist in the repo root (create from template if missing)
- `.sync-map.json` must exist in the repo root

## Workflow

### Phase 1: Discover Changes

1. Read `.sync-state.json` to get `last_synced_commit`
2. If `last_synced_commit` is null, use the first eve-horizon commit (full baseline sync)
3. Run in eve-horizon:
   ```bash
   cd ../eve-horizon && git log --oneline <last_synced_commit>..HEAD
   ```
4. Get the detailed diff of watched paths:
   ```bash
   cd ../eve-horizon && git diff --stat <last_synced_commit>..HEAD -- docs/system/ docs/ideas/agent-native-design.md docs/ideas/platform-primitives-for-agentic-apps.md packages/cli/src/commands/ AGENTS.md
   ```
5. For each changed file, read `git diff <last_synced_commit>..HEAD -- <file>` to understand what changed

### Phase 2: Analyze Impact

Using `.sync-map.json`, determine:

1. **Reference doc updates**: Which reference docs need updating based on changed source files
2. **Skill updates**: Which skills need updating based on changed concepts/commands
3. **New skill opportunities**: Are there new platform features that deserve their own skill?
4. **Refactoring opportunities**: Should any skills be split, merged, or restructured?

Create a change plan listing:
- Files to update with summary of changes
- New files to create with rationale
- Skills to refactor with reasoning

### Phase 3: Execute Updates

For each affected reference doc:
1. Read the current eve-horizon source doc(s)
2. Read the current skillpack reference doc
3. Distill the changes into the reference doc — these are curated distillations, not copies
4. Preserve the reference doc's existing structure and voice

For each affected skill:
1. Read the current skill SKILL.md
2. Identify what needs to change (new commands, changed workflows, new capabilities)
3. Update the skill maintaining imperative voice and conciseness

For new skills:
1. Create the skill directory with SKILL.md
2. Add references/ if needed
3. Update the parent pack README
4. Update ARCHITECTURE.md listings

### Phase 4: Update Tracking

1. Get the current HEAD of eve-horizon:
   ```bash
   cd ../eve-horizon && git rev-parse HEAD
   ```
2. Update `.sync-state.json`:
   - Set `last_synced_commit` to the HEAD hash
   - Set `last_synced_at` to current ISO timestamp
   - Append to `sync_log` (keep last 10 entries)
3. Update ARCHITECTURE.md if pack structure changed

### Phase 5: Report

Output a sync report:
```
## Sync Report: <old_commit_short>..<new_commit_short>

### Platform Changes
- <list of eve-horizon changes that affected skillpacks>

### Updated Reference Docs
- <file>: <what changed>

### Updated Skills
- <skill>: <what changed>

### New Skills
- <skill>: <why created>

### Refactoring
- <what was restructured and why>

### Next Steps
- <any manual follow-up needed>
```
