---
name: eve-new-project-setup
description: Set up a new Eve Horizon project from template - configures auth, profile, and manifest
triggers:
  - new project setup
  - initialize eve project
  - get started with eve
  - configure eve manifest
---

# Eve New Project Setup

You are helping a developer set up a new Eve Horizon project. They've just cloned the eve-horizon-starter template and need to configure it for their use.

## Prerequisites Check

First, verify their environment:

1. **Check for eve CLI**: Run `eve --version`. If not found, guide them to install:
   ```bash
   npm install -g @eve/cli
   ```

2. **Check for existing profile**: Run `eve profile show`. Look for a staging profile with api_url set.

3. **Check auth status**: Run `eve auth status`. See if they're authenticated.

## Step 1: Profile Setup (if needed)

If no staging profile exists:

```bash
eve profile create staging --api-url https://api.eve-staging.incept5.dev
```

Ask for their email and SSH key path, then set defaults:
```bash
eve profile set staging --default-email <their-email> --default-ssh-key <their-key-path>
```

## Step 2: Authentication (if needed)

If not authenticated:

```bash
eve auth bootstrap --status
```

- If window open and they're the first user → `eve auth bootstrap --email <email>`
- Otherwise → `eve auth login`
- If login fails, offer GitHub key auto-discovery

Verify with `eve auth status`.

## Step 3: Project Interview

Ask the user these questions to understand their project:

### Required Information
1. **Project name**: What should this project be called? (used for slug, e.g., `my-awesome-app`)
2. **Brief description**: One sentence about what this project does

### Eve Horizon Capabilities Discussion

Explain what Eve Horizon can help with:

> "Eve Horizon is a platform for running AI-powered jobs. Here's what it can do for your project:
>
> **CI/CD & Automation**
> - Run tests, builds, and deployments via AI agents
> - Automated code review and PR feedback
> - Release management and changelog generation
>
> **Development Workflows**
> - Code generation and scaffolding
> - Documentation generation
> - Dependency updates and security scanning
>
> **AI-Powered Tasks**
> - Natural language job definitions
> - Multi-step workflows with approvals
> - Secret management for API keys
>
> What aspects interest you most for this project?"

### Optional Configuration
3. **GitHub repo URL**: If they want GitHub integration (webhooks, PR comments)
4. **Default org**: If they're part of an org (check with `eve orgs`)

## Step 4: Configure Manifest

Read the current `.eve/manifest.yaml` file. Update it with the user's information:

```yaml
version: "1"
project:
  slug: <project-name-slug>  # lowercase, hyphens, from their project name
  name: <Project Name>
  description: <their description>

# If they provided GitHub URL:
integrations:
  github:
    repo: <owner/repo>
```

Write the updated manifest.

## Step 5: Git Configuration

Help them set up their own repo:

1. Check current remote: `git remote -v`
2. If it's still pointing to eve-horizon-starter:
   ```bash
   git remote rename origin upstream  # Keep template as upstream for updates
   git remote add origin <their-repo-url>
   ```
3. Update any template references (README, package.json name, etc.)

## Step 6: Verification & Next Steps

Run verification:
```bash
eve auth status        # Confirm authenticated
eve profile show       # Confirm profile configured
cat .eve/manifest.yaml # Confirm manifest updated
```

Provide next steps summary:
```
✅ Setup Complete!

Your project "<project-name>" is configured for Eve Horizon.

Next steps:
1. Push to your repo: git push -u origin main
2. Create your first job: eve jobs create --prompt "Hello Eve!"
3. Set up secrets: eve auth sync (syncs Claude/Codex tokens)
4. Explore workflows in .eve/workflows/

Need help? Run: eve --help
```

## Error Handling

- **No SSH key found**: Guide them to generate one or check GitHub for existing keys
- **Auth fails**: Check bootstrap status, suggest admin contact if needed
- **Manifest parse error**: Show the error, help fix YAML syntax
- **Git remote issues**: Provide manual commands to fix

## Conversation Style

Be friendly and efficient. Don't overwhelm with options - make smart defaults and ask for confirmation. The goal is to get them productive quickly.
