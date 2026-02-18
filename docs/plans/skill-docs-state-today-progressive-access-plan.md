# Skill Docs State-Today and Progressive Access Plan

> **Status**: Completed  
> **Last Updated**: 2026-02-18  
> **Intent**: Keep eve-skillpacks documentation aligned to shipped Eve Horizon behavior only, and make skill docs easier for agents to load progressively by intent.

## Why This Exists

`../eve-horizon/docs/system` intentionally mixes **Current (Implemented)** and **Planned (Not Implemented)** sections.  
Without explicit filtering, planned content leaks into skillpack distillations and causes agents to suggest non-shipped capabilities.

At the same time, `eve-read-eve-docs` should be a task router, not a large monolith. Agents should load only what they need for the current task.

## Scope

- `eve-work/eve-read-eve-docs/SKILL.md`
- `eve-work/eve-read-eve-docs/references/*.md`
- Related skill docs that still include planned/future framing
- `private-skills/sync-horizon/SKILL.md` as the durable enforcement mechanism for future syncs

## Guiding Principles

1. **State-today fidelity**
   - Distill only shipped platform behavior.
   - Remove/ignore roadmap sections (`Planned`, `What's next`, future notes).
2. **Task-first routing**
   - Entry skills should route by intent and minimize context load.
3. **Progressive disclosure**
   - Keep SKILL.md focused on selection + execution guidance.
   - Keep deep detail in references, loaded only when needed.
4. **Source-of-truth anchoring**
   - Use `../eve-horizon` docs/CLI source as authoritative input.

## Findings From Audit

- Planned/future leakage existed in:
  - `eve-work/eve-read-eve-docs/SKILL.md`
  - `eve-work/eve-read-eve-docs/references/overview.md`
  - `eve-work/eve-read-eve-docs/references/events.md`
  - `eve-work/eve-read-eve-docs/references/pipelines-workflows.md`
  - `eve-work/eve-read-eve-docs/references/skills-system.md`
  - `eve-se/eve-pipelines-workflows/SKILL.md`
- `../eve-horizon/docs/system` currently retains mixed current/planned sections by design, so sync logic must enforce filtering every run.

## Completed in This Pass

1. **Removed planned/future framing from distilled docs**
   - Removed planned sections from:
     - `eve-work/eve-read-eve-docs/references/events.md`
     - `eve-work/eve-read-eve-docs/references/pipelines-workflows.md`
     - `eve-work/eve-read-eve-docs/references/skills-system.md`
   - Removed "What's next" + planned/current framing from:
     - `eve-work/eve-read-eve-docs/references/overview.md`

2. **Refactored `eve-read-eve-docs` for progressive access**
   - Updated description and usage guidance to state-today.
   - Added a task-based routing section:
     - `eve-work/eve-read-eve-docs/SKILL.md` (`## Task Router (Progressive Access)`).

3. **Cleaned remaining explicit planned wording in active skills**
   - Removed planned section from:
     - `eve-se/eve-pipelines-workflows/SKILL.md`
  - Adjusted checklist wording:
    - `eve-design/eve-fullstack-app-design/SKILL.md` (`planned` -> `defined`).

4. **Phase 1 implemented: normalize progressive-intent reference entry points**
   - Added `Use When`, `Load Next`, and `Ask If Missing` sections to all `eve-work/eve-read-eve-docs/references/*.md` files:
     - `agents-teams.md`
     - `builds-releases.md`
     - `cli.md`
     - `deploy-debug.md`
     - `events.md`
     - `gateways.md`
     - `harnesses.md`
     - `jobs.md`
     - `manifest.md`
     - `overview.md`
     - `pipelines-workflows.md`
     - `secrets-auth.md`
     - `skills-system.md`

5. **Hardened future sync behavior**
   - Updated `private-skills/sync-horizon/SKILL.md` with:
     - Output standards for state-today + progressive access
     - Worker rules to exclude planned/roadmap content
     - Mandatory compliance scan commands
     - Report sections for compliance and progressive-access updates
 
6. **Phase 2 implemented: split `cli.md` into task modules**
   - Added:
     - `eve-work/eve-read-eve-docs/references/cli-auth.md`
     - `eve-work/eve-read-eve-docs/references/cli-org-project.md`
     - `eve-work/eve-read-eve-docs/references/cli-jobs.md`
     - `eve-work/eve-read-eve-docs/references/cli-pipelines.md`
     - `eve-work/eve-read-eve-docs/references/cli-deploy-debug.md`
   - Updated `eve-work/eve-read-eve-docs/references/cli.md` to route intent-first to these modules.

7. **Phase 3 implemented: intent coverage matrix**
   - Added `## Intent Coverage Matrix` to `eve-work/eve-read-eve-docs/SKILL.md`.
   - Added minimum-reference mapping for auth, setup, jobs, build/deploy, recovery, and installs.

8. **Phase 4 implemented: automated compliance guard**
   - Added `private-skills/sync-horizon/scripts/check-state-today.sh`.
   - Updated `private-skills/sync-horizon/SKILL.md` to run the script in Phase 4 post-checks.

9. **Phase 5 implementation checkpoint**
  - Ran `private-skills/sync-horizon/scripts/check-state-today.sh` and confirmed compliance checks pass.
  - Executed full `/sync-horizon` checkpoint through commit `4fc1b70f` → `401d370` (12 commits, CLI + local stack runtime updates).
  - Updated `.sync-state.json` to `401d370cb35f1ec44b81a56358b9638e5e8de02b` and applied `eve-work/eve-read-eve-docs` updates.

## Completed Phases

### Phase 1: Normalize Reference Entry Points

Add consistent top sections to each `eve-read-eve-docs/references/*.md`:
- `Use When`
- `Load Next`
- `Ask If Missing`
 
**Goal**: Improve agent navigation so references can be loaded in minimal slices.

**Status**: Completed in this implementation pass.

### Phase 2: Split Large References by Task

Break high-volume references (starting with `references/cli.md`) into task modules (for example: auth, jobs, pipelines, deploy/debug, org/project setup).

**Goal**: Reduce overloading and improve precision for agent retrieval.

**Status**: Completed in this implementation pass.

### Phase 3: Intent Coverage Matrix

Add a matrix mapping common agent intents to minimum required references and expected outputs.

**Goal**: Make retrieval deterministic and reduce unnecessary doc reads.

**Status**: Completed in this implementation pass.

### Phase 4: Automated Compliance Guard

Add lightweight automation (script or CI check) to fail when:
- Planned/future markers reappear in distilled docs
- `Task Router (Progressive Access)` is missing from `eve-read-eve-docs/SKILL.md`

**Goal**: Prevent drift between manual cleanup efforts.

**Status**: Completed in this implementation pass.

### Phase 5: Validate Through Full Sync Run

Run a full `/sync-horizon` execution using updated private skill instructions and confirm:
- planned-content scans pass
- progressive-access checks pass
- sync report includes compliance status

**Status**: Compliance checks implemented and passing; full `/sync-horizon` checkpoint completed.

## Compliance Commands

```bash
./private-skills/sync-horizon/scripts/check-state-today.sh
rg -n "^## Task Router \\(Progressive Access\\)" eve-work/eve-read-eve-docs/SKILL.md
```

## Definition of Done

- `eve-read-eve-docs` and related skill docs contain no planned/future behavior guidance.
- `eve-read-eve-docs/SKILL.md` reliably routes by intent with minimal reference load.
- Sync workflow explicitly enforces state-today output and checks it after each run.
- The post-sync report includes compliance and progressive-access notes.

## Out of Scope (This Plan)

- Changing eve-horizon system docs to remove planned sections (source docs remain mixed by design).
- Re-architecting all skillpacks in one pass.
