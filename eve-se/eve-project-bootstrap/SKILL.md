---
name: eve-project-bootstrap
description: Bootstrap an Eve-compatible project with org and project setup, CLI profile defaults, repo linkage, and first deploy. Use when onboarding a new repo or environment to Eve.
---

# Eve Project Bootstrap

Use this flow to connect a repo to Eve and get the first deploy running.

## Set the API target

- Export `EVE_API_URL` for the target stack.
- Local k8s uses `http://api.eve.lvh.me` (no port-forward needed).

## Create org and project

- `eve org ensure <org-name>`
- `eve project ensure --name <name> --slug <slug> --repo-url <git-url> --branch main`
- Set defaults: `eve profile set --org <org-id> --project <proj-id>`

## Add the manifest

- Create `.eve/manifest.yaml` in the repo.
- Keep it as the source of truth for components and environments.
- Use the `eve-manifest-authoring` skill for structure details.

## First deploy

- Local k8s loop: build images, import to k3d, run `eve env deploy <project> <env> --tag local`.
- Registry deploy: ensure registry secrets exist, then `eve env deploy <project> <env>`.

## Verify

- Track the deploy job with `eve job list` and `eve job follow <id>`.
- Access apps via `http://{component}.{project}-{env}.lvh.me` or the manifest domain.

## Recursive skill distillation

- Record onboarding friction and missing steps as updates to this skill.
- Add a new eve-se skill if a distinct workflow repeats.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
