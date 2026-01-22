---
name: beads-task-management
description: Work tracking with Beads - setup, maintenance, task breakdown, and finding work
---

# Beads Task Management

This skill covers how to use Beads for AI-native issue tracking in this repository.

## Overview

Beads stores issues in `.beads/issues.jsonl` and syncs directly on the current branch (typically main). The SQLite database (`.beads/*.db`) is your local working state - it's gitignored and rebuilt from JSONL on clone.

## New Repository Setup

### Fresh Clone

After cloning a repo with beads configured:

```bash
bd prime                    # Import issues from JSONL, rebuild database
bd doctor                   # Verify setup is healthy
bd list                     # See available work
```

### Initializing Beads in a New Repo

```bash
bd init                     # Creates .beads/ structure
bd sync                     # Commit initial state to current branch
```

The config file `.beads/config.yaml` uses sensible defaults:
```yaml
# issue-prefix auto-detects from directory name if not set
# sync-branch not set = syncs to current branch (main)
```

## Health Checks

Run `bd doctor` regularly to verify setup:

```bash
bd doctor           # Check health
bd doctor --fix     # Auto-fix common issues
```

**Healthy output looks like:**
```
✓ 60 passed  ⚠ 2 warnings  ✖ 0 failed
```

**Common issues and fixes:**

| Issue | Fix |
|-------|-----|
| Sync Divergence | `bd sync` |
| Missing hooks | `bd doctor --fix` |
| Prefix mismatch | `rm .beads/*.db && bd import -i .beads/issues.jsonl` |
| Database corruption | `rm .beads/*.db && bd prime` |

## Daily Workflow

### Starting a Session

```bash
bd sync              # Pull latest from team
bd ready             # Show unblocked work available to claim
bd list              # Show all open issues
```

### Finding What to Work On

When asked to "do the next bit of work" or find work autonomously:

1. **Check for in-progress work first:**
   ```bash
   bd list --status=in_progress
   ```
   If you have work in progress, continue that.

2. **Find ready work (unblocked, unassigned):**
   ```bash
   bd ready
   ```

3. **Priority order:**
   - P0 (critical) > P1 (high) > P2 (medium) > P3 (low) > P4 (backlog)
   - Prefer tasks over epics (epics are containers)
   - Prefer unblocked work (no `blockedBy` dependencies)

4. **Claim and start:**
   ```bash
   bd show <id>                           # Read full description
   bd update <id> --status=in_progress    # Claim it
   ```

### During Work

```bash
bd update <id> --status=in_progress      # Mark as started
bd comment <id> "Found the issue in X"   # Add progress notes
bd create --title="Fix Y" --type=task    # Create discovered work
bd dep add <new-id> <parent-id>          # Link as subtask
```

All mutations auto-flush to `.beads/issues.jsonl` - no manual sync needed during work.

### Completing Work

```bash
bd close <id>                    # Mark complete (auto-flushes)
bd close <id> --reason="Done"    # With note
bd close <id1> <id2> <id3>       # Close multiple at once
```

**Close issues immediately** when work is done - don't batch at session end.

### Ending a Session

```bash
bd sync                          # Push changes to remote (current branch)
```

Note: `bd sync` commits and pushes to your current branch (typically main). Local changes are already persisted via auto-flush.

## Breaking Down Large Tasks

When facing a large task or epic, decompose it into actionable subtasks:

### 1. Create the Epic (Container)

```bash
bd create --title="Implement feature X" --type=epic --priority=2
# Returns: eve-horizon-abc
```

### 2. Break into Subtasks

Think about:
- What are the distinct deliverables?
- What can be done in parallel vs sequentially?
- What are the risk areas that need investigation first?

```bash
# Create subtasks
bd create --title="Research approach for X" --type=task
bd create --title="Implement core X logic" --type=task
bd create --title="Add tests for X" --type=task
bd create --title="Update docs for X" --type=task
```

### 3. Link Dependencies

```bash
# Subtask depends on epic (parent-child)
bd dep add eve-horizon-def eve-horizon-abc

# Task blocked by another task
bd dep add eve-horizon-ghi eve-horizon-def  # ghi waits for def
```

### 4. Verify Structure

```bash
bd show eve-horizon-abc    # See epic with children
bd blocked                 # See what's waiting on what
bd ready                   # See what can be started now
```

### Decomposition Guidelines

| Task Size | Action |
|-----------|--------|
| < 1 hour | Just do it, no subtasks needed |
| 1-4 hours | Single task, maybe split if distinct phases |
| 4+ hours | Definitely break down into subtasks |
| Multi-day | Create epic with child tasks |

**Good subtask characteristics:**
- Single clear deliverable
- Can be verified as done
- Minimal dependencies on other subtasks
- Fits in one focused session

## Syncing Issues

### Regular Sync (Every Session)

```bash
bd sync    # Sync issues with current branch
```

This:
1. Exports local DB to JSONL
2. Pulls latest from remote
3. 3-way merges changes
4. Commits and pushes back to remote

Issues are tracked directly on your working branch (typically main), so they're always visible alongside your code.

## Recursive skill distillation

- Capture new Beads workflows and edge cases here.
- Split out a new skill when a focused pattern keeps repeating.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
