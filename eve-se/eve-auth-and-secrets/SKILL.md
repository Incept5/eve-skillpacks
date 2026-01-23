---
name: eve-auth-and-secrets
description: Extract OAuth tokens from host, wire project secrets, and manage auth for Eve deployments
---

# Eve Auth and Secrets

Use this workflow to configure authentication and secrets for Eve projects.

## When to use

- Setting up a new development environment.
- OAuth tokens expired or need refresh.
- Configuring secrets for deployments.
- Troubleshooting auth failures in jobs or deploys.

## Extract OAuth tokens

Eve uses OAuth tokens from Claude Code and Codex for agent harnesses.

### macOS

Tokens are stored in the macOS Keychain:

```bash
# Extract and save to system-secrets.env.local
./bin/eh auth extract --save

# Extract only Claude credentials
./bin/eh auth extract --save --claude

# Check current auth status
./bin/eh auth check
```

### Linux

Check these locations:
- `~/.claude/.credentials.json`
- `~/.claude/credentials.json`
- `~/.config/claude/credentials.json`

Or use GNOME Keyring if available.

### Codex/OpenAI

Check `~/.code/auth.json` for Codex OAuth tokens.

## Project secrets

Set secrets via the Eve API:

```bash
# Set a secret for a project
eve secret set --project proj_xxx API_KEY "your-api-key"

# List project secrets (keys only, not values)
eve secret list --project proj_xxx

# Delete a secret
eve secret delete --project proj_xxx API_KEY
```

## Secret interpolation

Reference secrets in manifests with `${secret.KEY}`:

```yaml
components:
  api:
    env:
      DATABASE_URL: postgres://app:${secret.POSTGRES_PASSWORD}@db:5432/app
      API_KEY: ${secret.EXTERNAL_API_KEY}
```

## Local secrets file

For local development, create `.eve/secrets.yaml`:

```yaml
# .eve/secrets.yaml (gitignored!)
POSTGRES_PASSWORD: localdev
EXTERNAL_API_KEY: test-key-123
```

This file is used for local interpolation; production secrets come from the API.

## K8s secrets sync

After extracting tokens, sync to the k8s cluster:

```bash
# Sync secrets and restart workers
./bin/eh k8s secrets
```

## Common issues

- **Token expired**: Re-run `./bin/eh auth extract --save`.
- **Secret not found**: Check `eve secret list` and set missing keys.
- **Interpolation error**: Verify `${secret.KEY}` syntax and key existence.
- **Local secrets not loading**: Ensure `.eve/secrets.yaml` exists and is valid YAML.

## Recursive skill distillation

- Add new auth patterns (SSO, service accounts) as they emerge.
- Extract cloud-specific auth flows into dedicated skills if needed.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
