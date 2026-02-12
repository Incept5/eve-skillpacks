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
- Setting up identity providers or org invites

## Authentication

```bash
eve auth login
eve auth login --ttl 30                # custom token TTL (1-90 days)
eve auth status
```

### Challenge-Response Flow

Eve uses challenge-response authentication. The default provider is `github_ssh`:

1. Client sends SSH public key fingerprint
2. Server returns a challenge (random bytes)
3. Client signs the challenge with the private key
4. Server verifies the signature and issues a JWT

### Token Types

| Type | Issued Via | Use Case |
|------|-----------|----------|
| User Token | `eve auth login` | Interactive CLI sessions |
| Job Token | Worker auto-issued | Agent execution within jobs |
| Minted Token | `eve auth mint` | Bot/service accounts |

JWT payloads include `sub` (user ID), `org_id`, `scope`, and `exp`. Verify tokens via the JWKS endpoint: `GET /auth/jwks`.

### Permissions

Check what the current token can do:

```bash
eve auth permissions
```

Register additional identities for multi-provider access:

```bash
curl -X POST "$EVE_API_URL/auth/identities" -H "Authorization: Bearer $TOKEN" \
  -d '{"provider": "nostr", "external_id": "<pubkey>"}'
```

## Identity Providers

Eve supports pluggable identity providers. The auth guard tries Bearer JWT first, then provider-specific request auth.

| Provider | Auth Method | Use Case |
|----------|------------|----------|
| `github_ssh` | SSH challenge-response | Default CLI login |
| `nostr` | NIP-98 request auth + challenge-response | Nostr-native users |

### Nostr Authentication

Two paths:
- **Challenge-response**: Like SSH but signs with Nostr key. Use `eve auth login --provider nostr`.
- **NIP-98 request auth**: Every API request signed with a Kind 27235 event. Stateless, no stored token.

## Org Invites

Invite external users by identity hint:

```bash
# Admin creates invite targeting a Nostr pubkey
curl -X POST "$EVE_API_URL/auth/invites" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"org_id": "org_xxx", "role": "member", "provider_hint": "nostr", "identity_hint": "<pubkey>"}'
```

When the pubkey authenticates, Eve auto-provisions their account and org membership.

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

Print the current access token (useful for scripts):

```bash
eve auth token
```

## Self-Service Access Requests

Users without an invite can request access:

```bash
eve auth request-access --org "My Company" --email you@example.com
eve auth request-access --org "My Company" --ssh-key ~/.ssh/id_ed25519.pub
eve auth request-access --status <request_id>
```

Admins approve or reject via:

```bash
eve admin access-requests list
eve admin access-requests approve <request_id>
eve admin access-requests reject <request_id> --reason "..."
```

## Credential Check

Verify local AI tool credentials:

```bash
eve auth creds                # Show Claude + Codex cred status
eve auth creds --claude       # Only Claude
eve auth creds --codex        # Only Codex
```

## OAuth Token Sync

Sync local OAuth tokens into Eve secrets (scope: project > org > user):

```bash
eve auth sync                       # Sync to user-level
eve auth sync --org org_xxx         # Sync to org-level
eve auth sync --project proj_xxx    # Sync to project-level
eve auth sync --dry-run             # Preview without syncing
```

## Key Rotation

Rotate the JWT signing key:

1. Set `EVE_AUTH_JWT_SECRET_NEW` alongside the existing secret
2. Server starts signing with the new key but accepts both during the grace period
3. After grace period (`EVE_AUTH_KEY_ROTATION_GRACE_HOURS`), remove the old secret
4. Emergency rotation: set only the new key (immediately invalidates all existing tokens)

## Project Secrets

```bash
# Set a secret
eve secrets set API_KEY "your-api-key" --project proj_xxx

# List keys (no values)
eve secrets list --project proj_xxx

# Delete a secret
eve secrets delete API_KEY --project proj_xxx

# Import from file
eve secrets import .env --project proj_xxx
```

### Secret Interpolation

Reference secrets in `.eve/manifest.yaml` using `${secret.KEY}`:

```yaml
services:
  api:
    environment:
      API_KEY: ${secret.API_KEY}
```

### Manifest Validation

Validate that all required secrets are set before deploying:

```bash
eve manifest validate --validate-secrets    # check secret references
eve manifest validate --strict              # fail on missing secrets
```

### Local Secrets File

For local development, create `.eve/dev-secrets.yaml` (gitignored):

```yaml
secrets:
  default:
    API_KEY: local-dev-key
    DB_PASSWORD: local-password
  staging:
    DB_PASSWORD: staging-password
```

### Worker Injection

At job execution time, resolved secrets are injected as environment variables into the worker container. File-type secrets are written to disk and referenced via `EVE_SECRETS_FILE`. The file is removed after the agent process reads it.

### Git Auth

The worker uses secrets for repository access:
- **HTTPS**: `github_token` secret → `Authorization: Bearer` header
- **SSH**: `ssh_key` secret → written to `~/.ssh/` and used via `GIT_SSH_COMMAND`

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Not authenticated | Run `eve auth login` |
| Token expired | Re-run `eve auth login` (tokens auto-refresh if within 5 min of expiry) |
| Secret missing | Confirm with `eve secrets list` and set the key |
| Interpolation error | Verify `${secret.KEY}` spelling; run `eve manifest validate --validate-secrets` |
| Git clone failed | Check `github_token` or `ssh_key` secret is set |
| Service can't reach API | Verify `EVE_API_URL` is injected (check `eve env show`) |

### Incident Response (Secret Leak)

If a secret may be compromised:
1. **Contain**: Rotate the secret immediately via `eve secrets set`
2. **Invalidate**: Redeploy affected environments
3. **Audit**: Check `eve job list` for recent jobs that used the secret
4. **Recover**: Generate new credentials at the source (GitHub, AWS, etc.)
5. **Document**: Record the incident and update rotation procedures
