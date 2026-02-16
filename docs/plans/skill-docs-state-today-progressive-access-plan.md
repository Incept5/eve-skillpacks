# Skill Docs State-Today and Progressive Access Plan

> **Status**: In Progress  
> **Last Updated**: 2026-02-16  
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

4. **Hardened future sync behavior**
   - Updated `private-skills/sync-horizon/SKILL.md` with:
     - Output standards for state-today + progressive access
     - Worker rules to exclude planned/roadmap content
     - Mandatory compliance scan commands
     - Report sections for compliance and progressive-access updates

## Next Phases

### Phase 1: Normalize Reference Entry Points

Add consistent top sections to each `eve-read-eve-docs/references/*.md`:
- `Use When`
- `Load Next`
- `Ask If Missing`

**Goal**: Improve agent navigation so references can be loaded in minimal slices.

### Phase 2: Split Large References by Task

Break high-volume references (starting with `references/cli.md`) into task modules (for example: auth, jobs, pipelines, deploy/debug, org/project setup).

**Goal**: Reduce overloading and improve precision for agent retrieval.

### Phase 3: Intent Coverage Matrix

Add a matrix mapping common agent intents to minimum required references and expected outputs.

**Goal**: Make retrieval deterministic and reduce unnecessary doc reads.

### Phase 4: Automated Compliance Guard

Add lightweight automation (script or CI check) to fail when:
- Planned/future markers reappear in distilled docs
- `Task Router (Progressive Access)` is missing from `eve-read-eve-docs/SKILL.md`

**Goal**: Prevent drift between manual cleanup efforts.

### Phase 5: Validate Through Full Sync Run

Run a full `/sync-horizon` execution using updated private skill instructions and confirm:
- planned-content scans pass
- progressive-access checks pass
- sync report includes compliance status

## Compliance Commands

```bash
rg -n "Planned \\(Not Implemented\\)|## Planned|What's next|current vs planned|Planned vs Current" eve-work/eve-read-eve-docs -g '*.md'
rg -n "^## Planned|Planned \\(Not Implemented\\)" eve-work eve-se eve-design -g 'SKILL.md'
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
