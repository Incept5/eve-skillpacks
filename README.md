# Eve Skillpacks

Public skill packs for Eve Horizon. Dev-only packs live in the eve-horizon repo under `private-eve-dev-skills/`.

## Install

Add to your `skills.txt`:
```
https://github.com/incept5/eve-skillpacks
```

This installs all skills in the repo for on-clone installs. For local installs, run:
```
skills add https://github.com/incept5/eve-skillpacks -a claude-code -y --all
# or add to skills.txt and run ./bin/eh skills install in an Eve repo
```

## Packs

- `eve-work` - Productive work patterns (includes `eve-orchestration`)
- `eve-se` - Platform-specific skills for deploys, manifests, pipelines, and agent runtime + chat ops
- `eve-design` - Architecture & design thinking (agent-native design principles)
