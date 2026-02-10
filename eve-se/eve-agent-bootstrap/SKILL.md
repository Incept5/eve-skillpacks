---
name: eve-agent-bootstrap
description: Bootstrap an Eve agent from zero — create profile, request access, wait for approval, set up project, and start building.
---

# Eve Agent Bootstrap

Self-service onboarding flow for agents on a fresh host. Goes from zero to a working Eve project.

## Step 1: Verify CLI

```bash
eve --version
```

If missing: `npm install -g @eve-horizon/cli`

## Step 2: Create Profile

```bash
eve profile create staging --api-url https://api.eh1.incept5.dev
eve profile use staging
```

## Step 3: Check Auth Status

```bash
eve auth status
```

- If **authenticated**: skip to Step 5.
- If **not authenticated**: continue to Step 4.

## Step 4: Request Access

Interview the user:
1. "Do you have an SSH key?" (default: `~/.ssh/id_ed25519.pub`)
2. "What do you want to call your org?" (e.g., "Acme Corp")
3. "What's your email?" (optional, for notifications)

Then submit and wait:

```bash
eve auth request-access \
  --ssh-key ~/.ssh/id_ed25519.pub \
  --org "Acme Corp" \
  --email agent@example.com \
  --wait
```

This blocks until an admin approves or rejects the request. On approval, the agent is automatically logged in.

**Admin side** (separate terminal or admin agent):

```bash
eve admin access-requests          # list pending
eve admin access-requests approve <areq_id>
```

## Step 5: Set Profile Defaults

After login, capture the org and project IDs:

```bash
eve org list --json
eve profile set --org <org_id>
```

## Step 6: Create Project (if needed)

Interview: project name, slug, repo URL.

```bash
eve project ensure --name "My App" --slug my-app \
  --repo-url https://github.com/me/my-app --branch main
eve profile set --project <proj_id>
```

## Step 7: Create Manifest

If `.eve/manifest.yaml` doesn't exist, create a minimal one:

```yaml
schema: eve/compose/v2

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    x-eve:
      port: 3000
```

## Step 8: Learn the Platform

Read the Eve platform reference to understand capabilities:

```
https://web.incept5-evshow-staging.eh1.incept5.dev/llms
```

This covers all CLI commands, manifest syntax, and platform features.

## Step 9: Summary

Print current state:

```bash
eve auth status
eve org list
eve project list
```

Next steps:
- `eve env deploy staging --ref main --repo-dir .` — deploy
- `eve job create --title "First task" --description "..."` — create work
- `eve agents sync --ref main --repo-dir .` — sync agent configs
