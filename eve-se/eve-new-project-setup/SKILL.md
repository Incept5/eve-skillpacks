---
name: eve-new-project-setup
description: Configure a new Eve Horizon project after running eve init (profile, auth, manifest, and repo linkage).
triggers:
  - new project setup
  - initialize eve project
  - get started with eve
  - configure eve manifest
  - run eve-new-project-setup
---

# Eve New Project Setup

Use this after a developer has run `eve init` and needs to configure the project for Eve Horizon.

## Context

The user has already run:
```bash
npm install -g @eve-horizon/cli
eve init my-project
cd my-project
```

This skill handles the remaining setup: profile, authentication, org/project registration, manifest customization, and git remote configuration.

## Step 1: Verify CLI

```bash
eve --version
```

If this fails, the CLI wasn't installed. Have them run:
```bash
npm install -g @eve-horizon/cli
```

## Step 2: Profile Setup

Create a profile for the staging environment:

```bash
eve profile create staging --api-url https://api.eve-staging.incept5.dev
eve profile use staging
```

Ask the user for their email and set defaults:

```bash
eve profile set --default-email user@example.com
```

## Step 3: Authentication

Check current auth status:

```bash
eve auth status
```

If not authenticated, log in:

```bash
eve auth login
```

The CLI will guide them through SSH key discovery (can fetch from GitHub if needed).

Optional - sync OAuth tokens for agent harnesses:

```bash
eve auth sync
```

## Step 4: Org + Project

Ask the user for:
- **Organization name** (e.g., "my-company")
- **Project name** (e.g., "My App")
- **Project slug** (e.g., "my-app")
- **Repo URL** (e.g., "git@github.com:me/my-app.git")

Create or ensure they exist:

```bash
eve org ensure my-company
eve project ensure --name "My App" --slug my-app --repo-url git@github.com:me/my-app.git --branch main
```

Set as defaults in the profile:

```bash
eve profile set --org org_xxx --project proj_xxx
```

## Step 5: Manifest Configuration

The starter template includes `.eve/manifest.yaml`. Update it with the project details:

```yaml
schema: eve/compose/v2
project: my-app

services:
  api:
    build:
      context: apps/api
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

Key fields to customize:
- `project`: Match the project slug
- `services`: Define your app's services
- `x-eve.ingress`: Configure public access

## Step 6: Git Remote

The project starts with no remote. Help set one up:

```bash
git remote -v
git remote add origin git@github.com:user/my-app.git
git push -u origin main
```

## Step 7: Verification

Run these checks to confirm setup:

```bash
eve system health
eve auth status
eve profile show
```

## Next Steps

After setup is complete, suggest:

1. **Run locally**: `docker compose up --build`
2. **Set secrets**: `eve secrets set MY_KEY "value"`
3. **Deploy to staging**: `eve pipeline run deploy --env staging`
4. **Create a job**: `eve jobs create --prompt "Review the codebase"`

## Troubleshooting

### "eve: command not found"
```bash
npm install -g @eve-horizon/cli
```

### "Not authenticated"
```bash
eve auth login
```

### "No profile"
```bash
eve profile create staging --api-url https://api.eve-staging.incept5.dev
```
