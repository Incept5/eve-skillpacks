# Eve Skillpacks - Agent Instructions

Read CLAUDE.md for project context and conventions.

## Syncing with Eve Horizon

To update skillpacks from the latest eve-horizon changes, run:

```
/sync-horizon
```

This reads the eve-horizon git log since the last tracked sync, identifies changes that affect skillpacks, updates reference docs and skills, and records the new sync point.

### Manual Sync Check

To see what's changed without updating:

```bash
cd ../eve-horizon && git log --oneline $(cat ../eve-skillpacks/.sync-state.json | jq -r '.last_synced_commit')..HEAD -- docs/system/ docs/ideas/agent-native-design.md packages/cli/src/commands/ AGENTS.md
```
