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

## Directory Structure

```
eve-skillpacks/
├── ARCHITECTURE.md        # This file
├── eve-se/                # Platform-specific skills
│   ├── README.md
│   ├── eve-project-bootstrap/
│   │   └── SKILL.md
│   ├── eve-manifest-authoring/
│   │   └── SKILL.md
│   ├── eve-deploy-debugging/
│   │   └── SKILL.md
│   ├── eve-aws-provisioning/
│   │   └── SKILL.md
│   ├── eve-staging-operations/
│   │   └── SKILL.md
│   └── eve-pipelines-workflows/
│       └── SKILL.md
├── eve-work/              # General work patterns
│   ├── README.md
│   ├── eve-orchestration/
│   │   └── SKILL.md
│   ├── eve-job-lifecycle/
│   │   └── SKILL.md
│   ├── eve-job-debugging/
│   │   └── SKILL.md
│   └── eve-skill-distillation/
│       └── SKILL.md
└── eve-dev/               # Internal development
    ├── README.md
    ├── beads-task-management/
    │   └── SKILL.md
    ├── eve-dev-workflow/
    │   └── SKILL.md
    ├── eve-platform-debugging/
    │   └── SKILL.md
    ├── eve-docs-upkeep/
    │   └── SKILL.md
    └── eve-system-map/
        └── SKILL.md
```

## Skill Packs

### eve-se (Software Engineering)
Skills for working with the Eve Horizon platform and conforming to its patterns.

**Skills included**:
- **eve-project-bootstrap**: Bootstrap a repo, org, and project for Eve
- **eve-manifest-authoring**: Author and maintain the Eve manifest
- **eve-deploy-debugging**: Deploy and debug Eve-compatible apps
- **eve-aws-provisioning**: Provision AWS VPS (k3s) for the P0 baseline deployment
- **eve-staging-operations**: Staging environment setup, deployment, and troubleshooting
- **eve-pipelines-workflows**: Define and run pipelines and workflows

**Who should use**: Teams building applications on Eve Horizon.

### eve-work
Skills for doing productive work using Eve Horizon patterns.

**Skills included**:
- **eve-orchestration**: Orchestrate jobs via depth propagation, parallel decomposition, relations, and control signals
- **eve-job-lifecycle**: Create, manage, and review jobs and dependencies
- **eve-job-debugging**: Monitor and debug jobs via CLI
- **eve-skill-distillation**: Turn repeated patterns into reusable skills

**Who should use**: Anyone using Eve Horizon for knowledge work - whether software engineering, research, writing, or other domains.

### eve-dev
Internal skills for developing and maintaining the Eve Horizon platform itself.

**Skills included**:
- **beads-task-management**: Work tracking with Beads - setup, maintenance, task breakdown, and finding work
- **eve-dev-workflow**: Internal dev workflow, runtime options, and testing guardrails
- **eve-platform-debugging**: Debug jobs, provisioning, and platform services
- **eve-docs-upkeep**: Keep docs current and cross-linked
- **eve-system-map**: Architecture, docs, and code navigation map

**Who should use**: Eve Horizon platform developers and contributors.

## Installation

Add this repo to `skills.txt` to install all packs:

```txt
https://github.com/incept5/eve-skillpacks
```

This installs every skill in the repo. For selective installs, clone the repo and list only the packs/skills you want in `skills.txt`, then run:

```bash
eve-worker skills install
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
2. `eve-worker skills install` (or `./bin/eh skills install`) installs to `.agent/skills/`
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
eve-worker skills install
openskills list
openskills read <skill-name>
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

1. `./.agent/skills/` (project universal)
2. `~/.agent/skills/` (global universal)
3. `./.claude/skills/` (project Claude-specific)
4. `~/.claude/skills/` (global Claude-specific)

Project skills shadow global skills with the same name.

## Integration with Eve Horizon

When a job runs:
1. Worker clones the repository
2. `.eve/hooks/on-clone.sh` runs `./bin/eh skills install`
3. Skills are installed from `skills.txt` into `.agent/skills/`
4. Harness reads skills directly from `.agent/skills/`

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
