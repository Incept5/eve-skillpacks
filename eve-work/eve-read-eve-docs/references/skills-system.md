# Skills System (Current)

Eve Horizon installs skills via the `skills` CLI. Skills live in repositories and are installed at clone time.

## Key Files

- `skills.txt`: manifest of skill sources
- `.agent/skills/`: universal install target (gitignored)
- `.claude/skills/`: Claude-specific target (symlink when possible)
- `AGENTS.md`: optional repo-managed index (not auto-generated)
- `.eve/packs.lock.yaml`: resolved AgentPacks (preferred)

## Skill Format

`SKILL.md` frontmatter:

```yaml
---
name: my-skill
description: One-line summary
---
```

Use progressive disclosure:
- metadata (always available)
- SKILL.md body (loaded on use)
- `references/`, `scripts/`, `assets/` as needed

## skills.txt Rules

- One source per line.
- Blank lines and `#` comments ignored.
- Local paths should be **explicit** (`./`, `../`, `/`, `~`).

Example:

```txt
./private-eve-dev-skills/eve-dev/beads-task-management
https://github.com/incept5/eve-skillpacks
```

## AgentPacks (Preferred)

AgentPacks are declared in `.eve/manifest.yaml` under `x-eve.packs` and resolved
into `.eve/packs.lock.yaml` by `eve agents sync`.

```bash
eve packs status
eve packs resolve --dry-run
```

## Install Flow

```bash
./bin/eh skills install
```

Installer actions:
1) Install each source via the `skills` CLI -> `.agent/skills/`
2) Symlink `.claude/skills` -> `.agent/skills` when possible

Workers run `.eve/hooks/on-clone.sh` to install skills on fresh clones.

## Using Skills

```bash
skill read <skill-name>
```

This prints the SKILL plus a base directory for resolving references.
