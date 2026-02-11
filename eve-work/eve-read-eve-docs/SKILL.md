---
name: eve-read-eve-docs
description: Load first. Index of distilled Eve Horizon system docs for CLI usage, manifests, pipelines, jobs, secrets, agents, builds, events, and debugging.
triggers:
  - eve docs
  - eve horizon docs
  - read eve docs
  - eve cli
  - eve manifest
  - eve pipelines
  - eve workflows
  - eve job
  - eve secrets
  - eve auth
  - eve events
  - eve triggers
  - eve agents
  - eve teams
  - eve builds
  - eve releases
  - eve deploy
---

# Eve Read Docs (Load First)

Purpose: provide a compact, public, always-available distillation of Eve Horizon system docs. Use this when private system docs are not accessible.

## When to Use

- Any question about how to use Eve Horizon via CLI or API.
- Any question about `.eve/manifest.yaml`, pipelines, workflows, jobs, or secrets.
- Any question about events, triggers, agents, teams, builds, or deployments.
- Any time you need the authoritative "current vs planned" status.

## How to Use

1. Start with `references/overview.md` for core concepts, IDs, and the reference index.
2. Open the relevant reference(s) below based on the task.
3. Prefer **Current (Implemented)** guidance; **Planned** sections are not live.

## Index

- `references/overview.md` -- Architecture, core concepts, IDs, job phases, reference index.
- `references/cli.md` -- CLI quick reference: all commands by category with flags and options.
- `references/manifest.md` -- Manifest v2 spec: services, environments, pipelines, workflows, x-eve extensions.
- `references/events.md` -- **Event type catalog** (all sources + payloads) and **trigger syntax** (github, slack, system, cron, manual).
- `references/jobs.md` -- Job lifecycle, phases, CLI, git/workspace controls, scheduling hints.
- `references/builds-releases.md` -- Build system (specs, runs, artifacts), releases, deploy model, promotion patterns.
- `references/agents-teams.md` -- Agent/team/chat YAML schemas, sync flow, slug rules, dispatch modes, coordination threads.
- `references/pipelines-workflows.md` -- Pipeline steps, triggers, workflow invocation, build-release-deploy pattern.
- `references/secrets-auth.md` -- Secrets scopes, interpolation, auth model, identity providers, OAuth sync, service principals, access visibility, custom roles, policy-as-code.
- `references/skills-system.md` -- Skills format, skills.txt, install flow, discovery priority.
- `references/deploy-debug.md` -- K8s architecture, worker images, deploy polling, ingress/TLS, secrets provisioning, workspace janitor, CLI debugging workflows, real-time debugging, env-specific debugging.
- `references/harnesses.md` -- Harness selection, profiles, auth priority, sandbox flags.
- `references/gateways.md` -- Gateway plugin architecture, Slack + Nostr providers, thread keys.

## Hard Rules

- Eve is **API-first**; the CLI only needs `EVE_API_URL`.
- Do **not** assume URLs, ports, or environment state--ask if unknown.
- If anything is missing or unclear, ask for the missing inputs.
