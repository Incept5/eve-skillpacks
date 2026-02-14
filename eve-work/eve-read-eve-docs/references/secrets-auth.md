# Secrets + Auth Reference

## Secrets

### Scope Hierarchy

Secrets resolve in priority order: **project > user > org > system**. Values are encrypted at rest and never returned in plaintext. The `show` endpoint returns a masked value (first/last characters only).

API paths: `/system/secrets`, `/users/:id/secrets`, `/orgs/:id/secrets`, `/projects/:id/secrets`.

### CLI

```bash
eve secrets list --project proj_xxx
eve secrets set KEY value --project proj_xxx
eve secrets show KEY --project proj_xxx

eve secrets ensure --project proj_xxx --keys GITHUB_WEBHOOK_SECRET
eve secrets export --project proj_xxx --keys GITHUB_WEBHOOK_SECRET
```

`ensure` auto-generates allowlisted secrets (e.g., `GITHUB_WEBHOOK_SECRET`). `export` returns the plaintext value for external configuration -- treat it as sensitive.

### Import Secrets from File

Batch-import secrets from a `KEY=VALUE` env file:

```bash
eve secrets import --org org_xxx --file ./secrets.env
eve secrets import --project proj_xxx --file .env
```

Supported scopes: `--project`, `--org`, `--user`, `--system` (admin only). Lines starting with `#` and blank lines are ignored. Values are read verbatim after `=` (quotes are not stripped). Each key is upserted as `env_var`.

### Manifest Validation

Declare required secrets in `eve.yaml`:

```yaml
x-eve:
  requires:
    secrets: [GITHUB_TOKEN, REGISTRY_TOKEN]
```

Validate with:
```bash
eve project sync --validate-secrets
eve project sync --strict
eve secrets validate --project proj_xxx
```

Validation reports missing secrets with scope-aware remediation hints.

### Interpolation

Reference secrets in manifest environment blocks:

```yaml
environment:
  DATABASE_URL: postgres://user:${secret.DB_PASSWORD}@db:5432/app
  API_KEY: ${secret.EXTERNAL_API_KEY}
```

### Local Dev Secrets

Create `.eve/dev-secrets.yaml` (gitignored) for local overrides:

```yaml
secrets:
  default:
    DB_PASSWORD: dev_password
  staging:
    DB_PASSWORD: staging_password
```

API secrets overlaid by local dev-secrets. Local takes precedence. For k8s production, set secrets via the API.

### Host Env Files

Two host-level files for local development (never committed):

- **`.env`** (repo root) -- Local secrets and internal tokens.
- **`system-secrets.env.local`** -- System-level defaults, bootstrapped on API startup. Restart API to pick up changes.

### Required System Vars

| Variable | Where | Purpose |
|----------|-------|---------|
| `EVE_SECRETS_MASTER_KEY` | API | Encryption key for secrets at rest |
| `EVE_INTERNAL_API_KEY` | Worker + API | Internal token for resolve endpoint |

### Worker Injection

- Resolved secrets are injected as **environment variables** for the worker and deployer (allowlisted).
- File and `ssh_key` secrets are written outside the repo workspace and are not available to agent processes.
- The worker does **not** write `.eve/secrets.env` into the workspace and does **not** set `EVE_SECRETS_FILE`.

### Git Auth Injection

- **HTTPS clone**: `github_token` secrets are injected into the clone URL for private repo access.
- **SSH clone**: `ssh_key` secrets are written to a temp key and wired via `GIT_SSH_COMMAND`.
- Missing auth surfaces explicit errors with remediation hints (`eve secrets set`).

### Fail-Fast on Resolution Failure

The worker fails fast on secret resolution failure instead of silently substituting
empty strings. Provider credentials are resolved at the platform layer, not cached
in the worker. If `EVE_INTERNAL_API_KEY` is missing or incorrect, the attempt fails
immediately with a descriptive error.

### Troubleshooting Secret Resolution

If a job fails during clone or secret resolution:

1. Confirm the secret exists: `eve secrets show <KEY> --project <id>`
2. Ensure `EVE_INTERNAL_API_KEY` and `EVE_SECRETS_MASTER_KEY` are set for API/worker
3. Check orchestrator/worker logs for `[resolveSecrets]` warnings
4. Re-run with corrected secret scope (project > org > system)

### Incident Response

If you suspect a secret leak:

1. **Contain** -- Rotate the affected secret at the source. Update via `eve secrets set` or `eve secrets import`.
2. **Invalidate** -- Restart affected services to flush cached credentials. If a token was printed to logs, assume compromise.
3. **Audit** -- Review job and pipeline logs for leakage patterns. Check correlation IDs.
4. **Recover** -- Re-run failed jobs after rotation.
5. **Document** -- Record what leaked, where, and why. Add guardrails if due to missing redaction.

---

## Auth

Eve uses **RS256 JWT** tokens with pluggable identity providers (SSH, Nostr). Supabase (HS256) mode is optional when `SUPABASE_JWT_SECRET` is set.

### Configuration

| Variable | Required | Description |
|----------|----------|-------------|
| `EVE_AUTH_ENABLED` | No | Enable auth (default `true`) |
| `EVE_AUTH_PRIVATE_KEY` | Yes | RSA private key (PEM string or file path) |
| `EVE_BOOTSTRAP_TOKEN` | Prod | One-time token for initial admin creation |
| `EVE_AUTH_PUBLIC_KEY` | No | Derived from private if omitted |
| `EVE_AUTH_CHALLENGE_TTL_SECONDS` | No | Challenge validity (default `300`) |
| `EVE_AUTH_TOKEN_TTL_DAYS` | No | User token TTL in days (default `1`, max `90`) |
| `EVE_AUTH_KEY_ID` | No | Key identifier in JWKS (default `key-1`) |

Generate keys:
```bash
openssl genrsa -out eve-auth.key 2048
openssl rsa -in eve-auth.key -pubout -out eve-auth.pub
export EVE_AUTH_PRIVATE_KEY="$(cat eve-auth.key)"
```

### Bootstrap

Create the first admin user. Three security modes:

| Mode | Trigger | Token Required |
|------|---------|----------------|
| **auto-open** | Fresh deploy, no users | No (10-min window) |
| **recovery** | Trigger file on host (`/tmp/eve-bootstrap-enable`) | No |
| **secure** | `EVE_BOOTSTRAP_TOKEN` set | Yes |

```bash
eve auth bootstrap --email admin@example.com --token $EVE_BOOTSTRAP_TOKEN
eve auth bootstrap --status
```

Production requires `EVE_BOOTSTRAP_TOKEN`. Use your real email -- it becomes your login identity.
In local/non-production environments, `eve auth bootstrap` attempts the API recovery path even when bootstrap is marked completed, and can return an existing admin token when recovery mode is allowed.

### Challenge-Response Login

Users authenticate by signing a challenge with a registered identity. The server selects the appropriate verifier based on identity type.

**CLI (recommended):**
```bash
eve auth login --email you@example.com
eve auth login --email you@example.com --ttl 30
```

The CLI requests a challenge, signs the nonce with your SSH key, submits the signature, and stores the token in `~/.eve/credentials.json`.

**SSH manual flow (curl):**
```bash
# 1. Request challenge
curl -X POST "$EVE_API_URL/auth/challenge" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'
# Response: { "challenge_id": "...", "nonce": "...", "expires_at": "..." }

# 2. Sign the nonce (namespace must be "eve-auth")
echo -n "$NONCE" | ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n eve-auth

# 3. Verify signature
curl -X POST "$EVE_API_URL/auth/verify" \
  -H "Content-Type: application/json" \
  -d '{"challenge_id": "...", "signature": "-----BEGIN SSH SIGNATURE-----\n..."}'
# Response: { "access_token": "...", "user_id": "...", "expires_at": ... }
```

**Nostr manual flow (curl):**
```bash
# 1. Request challenge (note: provider + pubkey, not email)
curl -X POST "$EVE_API_URL/auth/challenge" \
  -H "Content-Type: application/json" \
  -d '{"provider": "nostr", "pubkey": "<64-char-hex-pubkey>"}'

# 2. Sign a kind-22242 event with ["challenge", "<nonce>"] tag (client-side)

# 3. Submit signed event
curl -X POST "$EVE_API_URL/auth/verify" \
  -H "Content-Type: application/json" \
  -d '{"challenge_id": "...", "signature": "<kind-22242-event-json>"}'

# With invite code (for unregistered pubkeys):
curl -X POST "$EVE_API_URL/auth/verify" \
  -d '{"challenge_id": "...", "signature": "...", "invite_code": "abc123..."}'
```

### Self-Service Access Requests

Users can submit access requests without an invite:

```bash
eve auth request-access --org "My Company" --email you@example.com
eve auth request-access --org "My Company" --ssh-key ~/.ssh/id_ed25519.pub
eve auth request-access --status <request_id>
```

Admins review requests via:

```bash
eve admin access-requests list
eve admin access-requests approve <request_id>
eve admin access-requests reject <request_id> --reason "..."
```

Approval semantics:
- Approval is transactional (no partial org/user leftovers on failure).
- Duplicate fingerprints reuse the existing identity owner instead of failing.
- Re-approving an already-approved request is idempotent.
- Legacy partial orgs with matching slug+name are reused during approval.

### Credential Check

Inspect local AI tool credential availability:

```bash
eve auth creds                # Show Claude + Codex cred status
eve auth creds --claude       # Only Claude
eve auth creds --codex        # Only Codex
```

### OAuth Token Sync

Sync local OAuth tokens into Eve secrets:

```bash
eve auth sync                 # Sync to user-level (default)
eve auth sync --org org_xxx   # Sync to org-level
eve auth sync --project proj_xxx  # Sync to project-level
eve auth sync --dry-run       # Preview without syncing
```

### Token Types

**User Tokens** -- Issued on successful login. Used for API access.
```json
{ "sub": "user_abc123", "email": "user@example.com", "type": "user", "iat": 1706000000, "exp": 1706086400 }
```

**Job Tokens** -- Scoped tokens issued to running jobs with limited permissions.
```json
{ "user_id": "user_abc123", "org_id": "org_xyz789", "permissions": ["job:read", "job:write"], "type": "job", "iat": 1706000000, "exp": 1706086400 }
```

**Service Principal Tokens** -- RS256 JWT with `sub: sp:{principal_id}`, `type: service_principal`, explicit `scopes` array. No implicit role expansion.

### Permissions

Eve uses a unified permission model. Job tokens carry a limited `permissions` list scoped to the project/job. Custom roles are additive only.

```bash
eve auth permissions          # Full permission catalog
eve auth whoami               # Current user + effective permissions
```

API: `GET /auth/permissions` (catalog), `GET /auth/me` (current user).

Permission resolution: `effective = expand(base_role) UNION all(bound_custom_role_permissions)`.

### Identity Management

Identities can be SSH public keys, Nostr pubkeys, or other provider-specific credentials. Each is linked to a user account.

Register additional identities:
```bash
curl -X POST "$EVE_API_URL/auth/identities" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"public_key": "ssh-ed25519 AAAA... user@laptop", "label": "laptop"}'
```

Admins can register keys for other users by including `"email": "user@example.com"` in the body.

### JWKS Endpoint

Public keys for token verification:
```bash
curl "$EVE_API_URL/.well-known/jwks.json"
# Returns: { "keys": [{ "kty": "RSA", "kid": "key-1", "use": "sig", "alg": "RS256", ... }] }
```

During key rotation, both old and new keys appear in JWKS.

### Key Rotation

**Standard rotation (with grace period):**
```bash
# 1. Generate new key pair
openssl genrsa -out eve-auth-new.key 2048
openssl rsa -in eve-auth-new.key -pubout -out eve-auth-new.pub

# 2. Configure grace period
export EVE_AUTH_PRIVATE_KEY="$(cat eve-auth-new.key)"
export EVE_AUTH_PUBLIC_KEY="$(cat eve-auth-new.pub)"
export EVE_AUTH_PUBLIC_KEY_OLD="$(cat eve-auth-old.pub)"
export EVE_AUTH_KEY_ID="key-2"
export EVE_AUTH_KEY_ID_OLD="key-1"

# 3. Restart API. New tokens use key-2; old tokens still verify via key-1.
# 4. After old tokens expire (default 24h), remove OLD vars and restart.
```

**Emergency rotation (key compromise):**
```bash
export EVE_AUTH_PRIVATE_KEY="$(cat eve-auth-new.key)"
export EVE_AUTH_KEY_ID="key-emergency-$(date +%s)"
unset EVE_AUTH_PUBLIC_KEY_OLD   # Do not honor old tokens
# Restart. All existing tokens are immediately invalidated.
```

---

## Identity Providers

Eve uses a pluggable identity provider framework. Providers register at startup and the auth guard evaluates them in a two-stage chain: **Bearer JWT first**, then provider-specific request auth.

### Provider Interface

Every provider implements:

```typescript
interface IdentityProvider {
  readonly name: string;
  createChallenge(params): Promise<ChallengeData>;
  verifyChallenge(challenge, proof, identities): Promise<VerifiedIdentity | null>;
  fingerprint(publicKey: string): Promise<string>;
  // Optional: per-request auth (no login required)
  extractFromRequest?(req): ExtractedCredential | null;
  verifyRequestCredential?(credential): Promise<VerifiedIdentity | null>;
}
```

`fingerprint` computes a deterministic dedup key to prevent duplicate registrations. `VerifiedIdentity` with `identity: null` triggers invite-gated provisioning.

### Auth Chain

```
Request --> @Public route? --> allow
        --> Stage 1: Bearer JWT (RS256/HS256) --> success: allow
        --> Stage 2: Provider request auth (registry iterates all providers) --> success: allow
        --> 401 Unauthorized
```

Stage 2 catches all errors and logs warnings. A broken provider does not cause a 500.

### SSH Provider (`github_ssh`)

- **Challenge**: Random 32-byte `base64url` nonce.
- **Verify**: Runs `ssh-keygen -Y verify` as subprocess. Iterates all registered SSH identities for the user until one matches.
- **Fingerprint**: `ssh-keygen -lf`, returns MD5 fingerprint (e.g., `MD5:ab:cd:...`).
- **Request-level auth**: Not supported. SSH requires the challenge/response flow.

### Nostr Provider (`nostr`)

Two authentication paths:

**Challenge/Verify (login)**: Client signs a kind-22242 event containing a `["challenge", "<nonce>"]` tag. Server verifies event ID + Schnorr signature (BIP-340), checks the challenge tag, matches pubkey to registered identities. Unregistered pubkeys trigger invite-gated provisioning.

**NIP-98 Request Auth**: Per-request auth via `Authorization: Nostr <base64>` header. Client creates a kind-27235 event with URL, method, and body hash tags. Server validates:
1. Event ID + Schnorr signature
2. `kind === 27235`
3. URL tag matches canonical request URL
4. Method tag matches request method
5. For non-GET: `payload` tag equals `sha256(body)`
6. Timestamp within +/-60s of server time
7. Replay protection via `auth_request_replays` table (120s TTL)

### Invite-Gated Provisioning

When an unregistered identity authenticates, the system attempts to provision via org invites.

**Create an invite (admin):**
```bash
curl -X POST "$EVE_API_URL/auth/invites" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"org_id": "org_xxx", "provider_hint": "nostr", "identity_hint": "<pubkey>", "role": "member", "expires_in_hours": 72}'
```

**Provisioning flow:**
1. Unregistered identity authenticates (challenge/verify or NIP-98)
2. Provider returns `VerifiedIdentity` with `identity: null`
3. Invite lookup: explicit `invite_code` takes priority, then `identity_hint` matching
4. Atomic provisioning: create user (synthetic email), create identity row, create org membership with invite role, mark invite used

**List invites:**
```bash
curl "$EVE_API_URL/auth/invites/org_xxx" -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Harness Credentials

Preferred secrets for agent harnesses: `ANTHROPIC_API_KEY` (Claude/mclaude/zai), `OPENAI_API_KEY` or `CODEX_AUTH_JSON_B64` (Codex/Code), `GEMINI_API_KEY` or `GOOGLE_API_KEY` (Gemini), `Z_AI_API_KEY` (Z.ai). For private repos: `GITHUB_TOKEN` (HTTPS) or `ssh_key` (SSH via `GIT_SSH_COMMAND`).

---

## Service Principals + Token Minting

App backends authenticate as services via scoped tokens:

```bash
eve auth create-service-account --name "pm-app-backend" --org org_xxx \
  --scopes "jobs:create,jobs:read,projects:read"
eve auth list-service-accounts --org org_xxx
eve auth revoke-service-account --name pm-app-backend --org org_xxx
```

API: `POST /orgs/:id/service-principals`, `POST .../service-principals/:sp_id/tokens` (mint), `DELETE .../tokens/:tok_id` (revoke).

Admins can also mint tokens without SSH login (useful for bots):

```bash
eve auth mint --email app-bot@example.com --org org_xxx --ttl 90
```

Creates user and membership if needed. TTL capped at server's `EVE_AUTH_TOKEN_TTL_DAYS`.
