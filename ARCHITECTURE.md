# Skill Packs Architecture

> **What**: Grouped bundles of related skills for distribution and management.
> **Why**: Organize skills by domain/purpose for easier discovery and installation.

## Overview

Skill packs are directories containing related skills grouped by purpose. Each pack organizes skills around a specific domain or workflow, making it easy to discover and install coherent sets of capabilities.

Key characteristics:
- Each pack is a top-level directory in this repo
- Each skill is a subdirectory containing a `SKILL.md` file
- Packs are installed by referencing this repo in `skills.txt`
- The repo is visible (not hidden) to serve as reference examples

Dev-only packs are hosted in the eve-horizon repo under `private-eve-dev-skills/` and are not published here.

## Directory Structure

```
eve-skillpacks/
├── ARCHITECTURE.md        # This file
├── CLAUDE.md              # Project instructions for agents
├── .sync-state.json       # Tracks last synced eve-horizon commit
├── .sync-map.json         # Maps eve-horizon sources to skillpack targets
├── .claude/skills/
│   └── sync-horizon/      # /sync-horizon slash command
│       └── SKILL.md
├── eve-design/            # Architecture & design thinking
│   ├── README.md
│   ├── eve-agent-native-design/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── eve-horizon-primitives.md
│   ├── eve-fullstack-app-design/
│   │   └── SKILL.md
│   └── eve-agentic-app-design/
│       └── SKILL.md
├── eve-se/                # Platform-specific skills
│   ├── README.md
│   ├── eve-se-index/
│   │   └── SKILL.md
│   ├── eve-bootstrap/
│   │   └── SKILL.md
│   ├── eve-cli-primitives/
│   │   └── SKILL.md
│   ├── eve-project-bootstrap/
│   │   └── SKILL.md
│   ├── eve-new-project-setup/
│   │   └── SKILL.md
│   ├── eve-manifest-authoring/
│   │   └── SKILL.md
│   ├── eve-auth-and-secrets/
│   │   └── SKILL.md
│   ├── eve-local-dev-loop/
│   │   └── SKILL.md
│   ├── eve-deploy-debugging/
│   │   └── SKILL.md
│   ├── eve-troubleshooting/
│   │   └── SKILL.md
│   ├── eve-pipelines-workflows/
│   │   └── SKILL.md
│   └── eve-repo-upkeep/
│       └── SKILL.md
└── eve-work/              # General work patterns
    ├── README.md
    ├── eve-read-eve-docs/
    │   ├── SKILL.md
    │   └── references/    # 13 distilled reference docs
    ├── eve-orchestration/
    │   └── SKILL.md
    ├── eve-job-lifecycle/
    │   └── SKILL.md
    ├── eve-job-debugging/
    │   └── SKILL.md
    ├── eve-skill-distillation/
    │   └── SKILL.md
    └── eve-agent-memory/
        ├── SKILL.md
        └── references/
            └── storage-primitives.md
```

## Skill Packs

### eve-design (Architecture & Design Thinking)
Skills for thinking about how to build on Eve Horizon — not just how to use it.

**Skills included**:
- **eve-agent-native-design**: Design principles — parity, granularity, composability, emergent capability
- **eve-fullstack-app-design**: Architect a full-stack app — manifest, services, DB, pipelines, deploys, secrets, observability
- **eve-agentic-app-design**: Layer agentic capabilities — agents, teams, inference, memory, events, chat, coordination

**Design progression**: principles → PaaS architecture → agentic architecture. Each skill builds on the previous.

**Who should use**: Anyone designing applications, APIs, or agent systems on Eve Horizon.

### eve-se (Software Engineering)
Skills for working with the Eve Horizon platform and conforming to its patterns.

**Skills included**:
- **eve-se-index**: Load this first; route to the right skill
- **eve-bootstrap**: Full onboarding flow from zero to working project
- **eve-cli-primitives**: Core CLI commands and capabilities
- **eve-new-project-setup**: Set up a new project from the starter
- **eve-project-bootstrap**: Bootstrap a repo, org, and project for Eve
- **eve-plan-implementation**: Execute plan documents using Eve jobs
- **eve-manifest-authoring**: Author and maintain the Eve manifest
- **eve-auth-and-secrets**: Authenticate and manage project secrets
- **eve-local-dev-loop**: Docker Compose local dev loop + staging handoff
- **eve-deploy-debugging**: Deploy and debug Eve-compatible apps
- **eve-pipelines-workflows**: Define and run pipelines and workflows
- **eve-troubleshooting**: CLI-first troubleshooting runbooks
- **eve-repo-upkeep**: Keep repos aligned with platform best practices

**Who should use**: Teams building applications on Eve Horizon.

### eve-work
Skills for doing productive work using Eve Horizon patterns.

**Skills included**:
- **eve-read-eve-docs**: Platform documentation gateway — 13 curated reference docs distilled from eve-horizon
- **eve-orchestration**: Orchestrate jobs via depth propagation, parallel decomposition, relations, and control signals
- **eve-job-lifecycle**: Create, manage, and review jobs and dependencies
- **eve-job-debugging**: Monitor and debug jobs via CLI
- **eve-skill-distillation**: Turn repeated patterns into reusable skills
- **eve-agent-memory**: Choose and combine storage primitives for agent memory — workspace, attachments, threads, org docs, filesystem, managed DB

**Who should use**: Anyone using Eve Horizon for knowledge work - whether software engineering, research, writing, or other domains.



## Installation

Add this repo to `skills.txt` to install all packs:

```txt
https://github.com/incept5/eve-skillpacks
```

This installs every skill in the repo. For selective installs, clone the repo and list only the packs/skills you want in `skills.txt`, then run:

```bash
eve skills install
# or, from Eve Horizon
./bin/eh skills install
```

If you have a local clone and want to install only a subset, you can reference paths directly:

```txt
./eve-work/*
./eve-work/eve-orchestration
```

The installation process:
1. `skills.txt` references the repo or local pack paths
2. `eve skills install` (or `./bin/eh skills install`) installs to `.agents/skills/`
3. Workers execute the installer on clone via `.eve/hooks/on-clone.sh`

## Creating a New Skill Pack

### 1. Create Pack Directory

```bash
mkdir -p <pack-name>
```

### 2. Add Pack README

Document the pack's purpose and included skills:

```markdown
# <Pack Name>

Brief description of the pack's domain and purpose.

## Skills Included

- **skill-name** - Brief description

## Installation

Add to your `skills.txt`:
```
https://github.com/incept5/eve-skillpacks
```

## Who Should Use This

Target audience and use cases.
```
```

### 3. Create Skill Directories

Each skill needs:
- `SKILL.md` (required) - Main skill instructions
- `references/` (optional) - Detailed documentation
- `QUICKREF.md` (optional) - Quick reference card
- `scripts/` (optional) - Executable utilities
- `assets/` (optional) - Templates, images

### 4. Install and Test

```bash
eve skills install
ls .agents/skills
skill read <skill-name>
```

## Skill Format

Each skill follows Anthropic's SKILL.md format with frontmatter:

```markdown
---
name: skill-name
description: Brief description with trigger phrases
---

# Skill Title

Instructions in imperative form (not second-person).

## When to Use

Load this skill when...

## Instructions

To accomplish X:
1. Step one
2. Step two

See references/guide.md for details.
```

### Guidelines

- **name**: Required, hyphen-case identifier
- **description**: Required, 1-2 sentences with trigger phrases
- **Body**: Imperative form ("Do X"), not "You should..."
- **Size**: Keep SKILL.md under 5,000 words; move details to `references/`
- **Resources**: Use `references/` for documentation, `scripts/` for utilities, `assets/` for templates

## Skill Discovery

Skills installed from this repo follow the standard OpenSkills search priority:

1. `./.agents/skills/` (project universal)
2. `~/.agents/skills/` (global universal)
3. `./.claude/skills/` (project Claude-specific)
4. `~/.claude/skills/` (global Claude-specific)

Project skills shadow global skills with the same name.

## Integration with Eve Horizon

When a job runs:
1. Worker clones the repository
2. `.eve/hooks/on-clone.sh` runs `./bin/eh skills install`
3. Skills are installed from `skills.txt` into `.agents/skills/`
4. Harness reads skills directly from `.agents/skills/`

No system-level skill configuration is needed. All skills come from the repository.

## Design Decisions

### Why Skill Packs?

- **Discovery**: Group related skills by domain for easier navigation
- **Distribution**: Install coherent sets of capabilities together
- **Organization**: Separate platform skills from user skills from dev tools
- **Versioning**: Track skill sources alongside code
- **Reference**: Visible repo serves as documentation-by-example for skill authoring

### Why a Dedicated Repo?

The packs live in a dedicated public repo so any project can reference them via `skills.txt`.

## Navigation

- Skills system: https://github.com/incept5/eve-horizon/blob/main/docs/system/skills.md
- Skills manifest: https://github.com/incept5/eve-horizon/blob/main/docs/system/skills-manifest.md
- Skillpacks user guide: https://github.com/incept5/eve-horizon/blob/main/docs/system/skillpacks.md
