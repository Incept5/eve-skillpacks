---
name: starter-upkeep
description: Keep eve-horizon-starter template in sync with eve-horizon-2 templates and platform conventions
---

# Starter Template Upkeep

Use this workflow to maintain the eve-horizon-starter template and ensure it stays current with platform changes.

## When to use

- After updating Eve CLI commands that affect starter workflows
- When manifest schema changes
- After adding new skillpacks that starters should use
- Periodically to catch deprecated patterns
- Before major releases

## Files involved

**Source of truth** (in eve-horizon-2):
- `templates/starter/` - Canonical template files

**Target** (sister repo):
- `../eve-horizon-starter` - Rendered starter repo for users

## Checking for drift

```bash
# In eve-horizon-2 directory
./bin/eh starter check   # CI-friendly, exits 0 if in sync
./bin/eh starter diff    # Show detailed differences
```

## Syncing changes

```bash
# Edit files in templates/starter/, then sync to starter repo
./bin/eh starter sync

# Then commit both repos:
cd ../eve-horizon-starter && git add -A && git commit -m "sync: update from templates"
```

## What to update

### Template files to keep current

1. **`.eve/manifest.yaml`** - Update when schema changes, add new best-practice examples
2. **`.eve/hooks/on-clone.sh`** - Update skill installation commands
3. **`skills.txt`** - Add new recommended skillpacks
4. **`README.md`** - Update getting started instructions, CLI command examples
5. **`CLAUDE.md`** - Update agent instructions for new workflows
6. **`scripts/*.sh`** - Update helper script commands

### Common updates

- CLI command renames or new flags
- New environment variables
- Changed default ports or URLs
- New skillpacks in eve-se
- Updated Docker base images

## CI integration

The `eh starter check` command is designed for CI:

```yaml
# .github/workflows/ci.yml
- name: Check starter drift
  run: ./bin/eh starter check
```

This fails the build if the starter repo has drifted from templates.

## Recursive skill distillation

- After making starter template changes, update this skill if the process changes
- If specific areas (manifest patterns, hook conventions) grow complex, extract into dedicated eve-se skills
- Update the eve-skillpacks README after adding new patterns
