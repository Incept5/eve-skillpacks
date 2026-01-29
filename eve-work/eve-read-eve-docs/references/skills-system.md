# Skills System (Current)

Eve Horizon uses **OpenSkills**. Skills live in repositories and are installed at clone time.

## Key Files

- `skills.txt`: manifest of skill sources
- `.agent/skills/`: universal install target (gitignored)
- `.claude/skills/`: Claude-specific target (symlink when possible)
- `AGENTS.md`: generated/updated by `openskills sync`

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

## Install Flow

```bash
./bin/eh skills install
```

Installer actions:
1) `openskills install <source> --universal` -> `.agent/skills/`
2) symlink `.claude/skills` -> `.agent/skills` when possible
3) `openskills sync` to update `AGENTS.md`

Workers run `.eve/hooks/on-clone.sh` to install skills on fresh clones.

## Using Skills

```bash
openskills read <skill-name>
```

This prints the SKILL plus a base directory for resolving references.
