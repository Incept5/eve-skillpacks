---
name: eve-auth-and-secrets
description: Authenticate with Eve and manage project secrets for deployments and workflows.
---

# Eve Auth and Secrets

Use this workflow to log in to Eve and manage secrets for your app.

## When to Use

- Setting up a new project profile
- Authentication failures
- Adding or rotating secrets
- Secret interpolation errors during deploys

## Authentication

```bash
eve auth login
eve auth login --ttl 30                # custom token TTL (1-90 days)
eve auth status
```

Optional (sync local OAuth tokens for agent harnesses):

```bash
eve auth sync
```

## Token Minting (Admin)

Mint tokens for bot/service users without SSH login:

```bash
# Mint token for a bot user (creates user + membership if needed)
eve auth mint --email app-bot@example.com --org org_xxx

# With custom TTL (1-90 days, default: server configured)
eve auth mint --email app-bot@example.com --org org_xxx --ttl 90

# Scope to project with admin role
eve auth mint --email app-bot@example.com --project proj_xxx --role admin
```

TTL is capped at the server's `EVE_AUTH_TOKEN_TTL_DAYS` env var (max 90 days).

## Project Secrets

```bash
# Set a secret
eve secrets set API_KEY "your-api-key" --project proj_xxx

# List keys (no values)
eve secrets list --project proj_xxx

# Delete a secret
eve secrets delete API_KEY --project proj_xxx
```

## Secret Interpolation

Reference secrets in `.eve/manifest.yaml` using `${secret.KEY}`:

```yaml
services:
  api:
    environment:
      API_KEY: ${secret.API_KEY}
```

## Local Secrets File

For local development, create `.eve/dev-secrets.yaml` (gitignored):

```yaml
secrets:
  default:
    API_KEY: local-dev-key
    DB_PASSWORD: local-password
  staging:
    DB_PASSWORD: staging-password
```

This file is used for local interpolation; production secrets come from the Eve API.

## Common Issues

- **Not authenticated**: run `eve auth login`.
- **Secret missing**: confirm with `eve secrets list` and set the key.
- **Interpolation error**: verify `${secret.KEY}` spelling and that the secret exists.
