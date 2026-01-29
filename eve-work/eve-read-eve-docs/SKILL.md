---
name: eve-read-eve-docs
description: Load first. Index of distilled Eve Horizon system docs for CLI usage, manifests, pipelines, jobs, secrets, and debugging.
triggers:
  - eve docs
  - eve horizon docs
  - read eve docs
  - eve cli
  - eve manifest
  - eve pipelines
  - eve workflows
  - eve jobs
  - eve secrets
  - eve auth
---

# Eve Read Docs (Load First)

Purpose: provide a compact, public, always-available distillation of Eve Horizon system docs. Use this when private system docs are not accessible.

## When to Use

- Any question about how to use Eve Horizon via CLI or API.
- Any question about `.eve/manifest.yaml`, pipelines, workflows, jobs, or secrets.
- Any time you need the authoritative "current vs planned" status.

## How to Use

1. Start with `references/overview.md` for core concepts and IDs.
2. Open the relevant reference(s) below based on the task.
3. Prefer **Current (Implemented)** guidance; **Planned** sections are not live.

## Index

- `references/overview.md` -- Architecture, core concepts, IDs, job phases.
- `references/cli.md` -- CLI quick reference, profiles, org/project, core commands.
- `references/manifest.md` -- Manifest v2 spec, services, envs, pipelines, workflows.
- `references/pipelines-workflows.md` -- Pipeline steps, triggers, workflow invocation.
- `references/jobs.md` -- Job lifecycle, job CLI, git/workspace controls.
- `references/secrets-auth.md` -- Secrets scopes, interpolation, auth + bootstrap.
- `references/skills-system.md` -- OpenSkills, skills.txt, install flow.
- `references/deploy-debug.md` -- Deploy modes, env deploy, CLI-first debugging.
- `references/harnesses.md` -- Harness selection, profiles, auth, sandbox.
- `references/events.md` -- Event spine and trigger routing.

## Hard Rules

- Eve is **API-first**; the CLI only needs `EVE_API_URL`.
- Do **not** assume URLs, ports, or environment state--ask if unknown.
- If anything is missing or unclear, ask for the missing inputs.
