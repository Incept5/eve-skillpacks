# Secrets + Auth (Current)

## Secrets Model

Scopes (highest wins): **project -> user -> org -> system**. Values are encrypted at rest and never returned in plaintext.

### CLI

```bash
eve secrets list --project proj_xxx
eve secrets set KEY value --project proj_xxx

eve secrets import --org org_xxx --file ./secrets.env

eve secrets validate --project proj_xxx

eve secrets ensure --project proj_xxx --keys GITHUB_WEBHOOK_SECRET
eve secrets export --project proj_xxx --keys GITHUB_WEBHOOK_SECRET
```

### Manifest Requirements

```yaml
x-eve:
  requires:
    secrets: [GITHUB_TOKEN, REGISTRY_TOKEN]
```

Use validation:
- `eve project sync --validate-secrets`
- `eve project sync --strict`
- `eve secrets validate --project proj_xxx`

### Interpolation

```yaml
environment:
  DATABASE_URL: postgres://user:${secret.DB_PASSWORD}@db:5432/app
```

### Local Dev Secrets

`.eve/dev-secrets.yaml` (gitignored) can provide local overrides:

```yaml
secrets:
  default:
    DB_PASSWORD: dev_password
  staging:
    DB_PASSWORD: staging_password
```

Local secrets override API secrets when using `file://` repos. In k8s production, **set secrets via API**.

### Required System Vars

- `EVE_SECRETS_MASTER_KEY` (API encryption key)
- `EVE_INTERNAL_API_KEY` (worker <-> API secret resolution)

## Auth Model

Default auth uses **RS256** JWTs with SSH challenge/response.
Supabase (HS256) is optional.

### Bootstrap (Admin)

```bash
eve auth bootstrap --email admin@example.com --token $EVE_BOOTSTRAP_TOKEN
```

Modes:
- **auto-open**: first 10 minutes on fresh install
- **recovery**: enable via trigger file on host
- **secure**: requires `EVE_BOOTSTRAP_TOKEN`

### Login

```bash
eve auth login --email you@example.com

# Custom token TTL (1-90 days, default: server configured)
eve auth login --email you@example.com --ttl 30
```

The CLI can auto-fetch SSH keys from GitHub if none are registered.

### Invite

```bash
eve admin invite --email user@example.com --github user
```

### Local Credentials

Check what AI tool credentials are available on your machine:

```bash
eve auth creds              # Show local Claude/Codex credentials
eve auth creds --claude     # Only check Claude
eve auth creds --codex      # Only check Codex
eve auth creds --json       # JSON output
```

Credential locations:
- **Claude Code**: macOS Keychain (`Claude Code-credentials`) or `~/.claude/.credentials.json`
- **Codex/Code**: macOS Keychain or `~/.codex/auth.json` / `~/.code/auth.json`

### OAuth Sync

Sync local OAuth tokens (Claude/Codex) to Eve:

```bash
eve auth sync                     # Sync to user-level (default)
eve auth sync --org org_xxx       # Sync to org-level
eve auth sync --project proj_xxx  # Sync to project-level
eve auth sync --dry-run           # Preview without syncing
eve auth sync --claude            # Only sync Claude tokens
eve auth sync --codex             # Only sync Codex tokens
```

Scope priority: `--project` > `--org` > user (default).
Default is user-level, so credentials are available to all your jobs.

## Identity Providers

Eve supports pluggable identity providers through a provider framework. The auth guard evaluates credentials in order: **Bearer JWT first**, then provider-specific request auth.

| Provider | Auth Method | Use Case |
|----------|------------|----------|
| `github_ssh` | SSH challenge-response | Default CLI login (`eve auth login`) |
| `nostr` | NIP-98 request auth + kind-22242 challenge-response | Nostr-native users |

New identities are **invite-gated**: an admin must create an invite targeting the identity before the provider can provision an account.

### CLI / API

```bash
# SSH login (default)
eve auth login --email you@example.com

# Nostr challenge (API-level)
# POST /auth/challenge {"provider": "nostr", "pubkey": "<hex>"}
```

## Org Invites

Admins can invite external users by identity hint. When the targeted identity authenticates, Eve auto-provisions their account and org membership.

```bash
# Create invite (admin)
curl -X POST "$EVE_API_URL/auth/invites" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"org_id": "org_xxx", "role": "member", "provider_hint": "nostr", "identity_hint": "<pubkey>"}'

# List invites
curl "$EVE_API_URL/auth/invites/org_xxx" -H "Authorization: Bearer $TOKEN"
```

## Service Principals (Machine Identity)

App backends authenticate as services (not users) via scoped tokens.

### CLI

```bash
# Create a service account + mint a token
eve auth create-service-account --name "pm-app-backend" --org org_xxx \
  --scopes "jobs:create,jobs:read,projects:read"

# List service accounts
eve auth list-service-accounts --org org_xxx

# Revoke a service account
eve auth revoke-service-account --name pm-app-backend --org org_xxx
```

### API

```
POST   /orgs/:id/service-principals              -- create
GET    /orgs/:id/service-principals              -- list
DELETE /orgs/:id/service-principals/:sp_id       -- delete
POST   /orgs/:id/service-principals/:sp_id/tokens -- mint scoped token
GET    /orgs/:id/service-principals/:sp_id/tokens  -- list tokens
DELETE /orgs/:id/service-principals/:sp_id/tokens/:tok_id -- revoke token
```

Token format: RS256 JWT with `sub: sp:{principal_id}`, `type: service_principal`, `scopes: [...]`. Scopes must come from the known permission catalog. `system:*` scopes require system admin.

Service principal tokens use the same permission guard path as job tokens — explicit scopes only, no implicit role expansion.

## Access Visibility

Query "can user X do Y in project Z?" or "why was this denied?"

### CLI

```bash
eve access can --org org_xxx --user user_abc --project proj_xxx --permission chat:write
# Output: ALLOWED (source: admin role on project proj_xxx)

eve access explain --org org_xxx --user user_abc --project proj_xxx --permission jobs:admin
# Output:
# Permission: jobs:admin
# Result: DENIED
# Grants found:
#   - org membership: member → [jobs:read, jobs:write] (missing jobs:admin)
# Missing: jobs:admin requires admin role or higher

# Also works for service principals
eve access can --org org_xxx --service-principal sp_xxx --permission jobs:read
```

### API

```
GET /orgs/:id/access/can?principal_type=user&principal_id=...&permission=...&project_id=...
GET /orgs/:id/access/explain?principal_type=user&principal_id=...&permission=...&project_id=...
```

Both require `orgs:admin`. Works for `user` and `service_principal` principal types.

## Custom Roles + Bindings

Additive role overlays on top of built-in member/admin/owner roles.

### CLI

```bash
# Create a custom role
eve access roles create pm_manager --org org_xxx --scope org \
  --permissions jobs:read,jobs:write,threads:read,threads:write,chat:write

eve access roles list --org org_xxx
eve access roles show pm_manager --org org_xxx
eve access roles update pm_manager --org org_xxx --add-permissions events:read
eve access roles delete pm_manager --org org_xxx

# Bind a role to a user or service principal
eve access bind --org org_xxx --project proj_xxx --user user_abc --role pm_manager
eve access bindings list --org org_xxx
eve access unbind --org org_xxx --project proj_xxx --user user_abc --role pm_manager
```

### API

```
POST/GET    /orgs/:id/access/roles              -- create/list roles
GET/PATCH/DELETE /orgs/:id/access/roles/:role_id -- get/update/delete role
POST/GET    /orgs/:id/access/bindings           -- create/list bindings
DELETE      /orgs/:id/access/bindings/:bind_id  -- delete binding
```

### Guardrails

- Custom roles are **additive only** — they cannot remove base membership grants.
- No explicit deny rules — only additive grants.
- Callers cannot bind roles with permissions they don't themselves hold.
- `system:*` permissions require system admin to grant.
- Permission resolution: `effective = expand(base_role) UNION all(bound_custom_role_permissions)`.

## Policy-as-Code (.eve/access.yaml)

Declare roles and bindings in YAML, sync to the API.

```yaml
version: 1
access:
  roles:
    pm_manager:
      scope: org
      permissions:
        - jobs:read
        - jobs:write
        - threads:read
        - threads:write
        - chat:write
    support_triage:
      scope: project
      permissions:
        - jobs:read
        - jobs:write
        - threads:read
        - events:read

  bindings:
    - scope: org
      subject:
        type: user
        id: user_pm123
      roles: [pm_manager]
    - scope: project
      project_id: proj_xxx
      subject:
        type: service_principal
        id: sp_pmapp
      roles: [support_triage]
```

### CLI

```bash
eve access validate --file .eve/access.yaml        # Schema + semantic validation
eve access plan --file .eve/access.yaml --org org_xxx  # Show diff (add/update/remove)
eve access sync --file .eve/access.yaml --org org_xxx  # Apply changes
eve access sync --file .eve/access.yaml --org org_xxx --prune  # Remove undeclared
eve access plan --file .eve/access.yaml --org org_xxx --json   # CI-friendly output
```

- `validate`: checks YAML schema, permission names, binding references
- `plan`: shows diff between declared (YAML) and actual (API) state
- `sync`: applies the plan (with confirmation unless `--yes`)
- `--prune`: removes roles/bindings in API but not in YAML
- Idempotent: running sync twice produces no changes

## Harness Credentials (Current)

Preferred secrets:
- **Claude / mclaude / zai**: `ANTHROPIC_API_KEY` (highest priority) or OAuth tokens
- **Codex / Code**: `OPENAI_API_KEY` or `CODEX_AUTH_JSON_B64`
- **Gemini**: `GEMINI_API_KEY` or `GOOGLE_API_KEY`
- **Z.ai**: `Z_AI_API_KEY`

For private repo access:
- `GITHUB_TOKEN` / `github_token` secrets are used for HTTPS clone.
- `ssh_key` secrets are used for SSH clone via `GIT_SSH_COMMAND`.

Troubleshooting:
- Missing git auth fails fast with remediation hints (`eve secrets set`).
- Check API/worker env for `EVE_INTERNAL_API_KEY` and `EVE_SECRETS_MASTER_KEY`.
