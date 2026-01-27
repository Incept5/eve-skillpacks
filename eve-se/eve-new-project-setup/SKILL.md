---
name: eve-new-project-setup
description: Set up a new Eve Horizon project from the starter template (CLI, profile, auth, manifest, and repo linkage).
triggers:
  - new project setup
  - initialize eve project
  - get started with eve
  - configure eve manifest
---

# Eve New Project Setup

Use this when a developer just cloned the starter template and needs it configured for Eve.

## Prerequisites Check

1. **Eve CLI installed**:
   ```bash
   eve --version
   ```
   If missing:
   ```bash
   npm install -g @eve-horizon/cli
   ```

2. **Profile exists**:
   ```bash
   eve profile show
   ```

3. **Auth status**:
   ```bash
   eve auth status
   ```

## Step 1: Profile Setup

Get the staging API URL from the admin and create a profile:

```bash
eve profile create staging --api-url https://api.eve-staging.incept5.dev
eve profile use staging
```

Set defaults:

```bash
eve profile set --default-email you@example.com --default-ssh-key ~/.ssh/id_ed25519
```

## Step 2: Authentication

```bash
eve auth login
eve auth status
```

Optional (for agent harnesses):

```bash
eve auth sync
```

## Step 3: Org + Project

Gather:

- Org name
- Project name and slug
- Repo URL

Create or ensure them:

```bash
eve org ensure my-org
eve project ensure --name "My App" --slug my-app --repo-url git@github.com:me/my-app.git --branch main
```

Set defaults:

```bash
eve profile set --org org_xxx --project proj_xxx
```

## Step 4: Manifest (v2)

If the repo uses an old `components:` manifest, migrate to `services:` and add
`schema: eve/compose/v1`.

Minimal example:

```yaml
schema: eve/compose/v1
project: my-app

services:
  api:
    image: ghcr.io/myorg/my-app-api:latest
    ports: [3000]
    x-eve:
      ingress:
        public: true
        port: 3000

environments:
  staging:
    pipeline: deploy

pipelines:
  deploy:
    steps:
      - name: deploy
        action: { type: deploy }
```

## Step 5: Git Remote

```bash
git remote -v
git remote set-url origin git@github.com:me/my-app.git
```

## Step 6: Verification + Next Steps

```bash
eve system health
eve auth status
eve profile show
```

Next steps:

1. Run locally with Docker Compose (`eve-local-dev-loop`)
2. Set secrets: `eve secrets set KEY "value" --project proj_xxx`
3. Deploy: `eve env deploy proj_xxx staging`
4. Create a job: `eve job create --description "Review the codebase"`
