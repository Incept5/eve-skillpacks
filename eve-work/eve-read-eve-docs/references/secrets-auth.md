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

`.eve/secrets.yaml` (gitignored) can provide local overrides:

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
```

The CLI can auto-fetch SSH keys from GitHub if none are registered.

### Invite

```bash
eve admin invite --email user@example.com --github-username user
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

## Harness Credentials (Current)

Preferred secrets:
- **Claude / mclaude / zai**: `ANTHROPIC_API_KEY` (highest priority) or OAuth tokens
- **Codex / Code**: `OPENAI_API_KEY` or `CODEX_AUTH_JSON_B64`
- **Gemini**: `GEMINI_API_KEY` or `GOOGLE_API_KEY`
- **Z.ai**: `Z_AI_API_KEY`

For private repo access:
- `GITHUB_TOKEN` or `ssh_key` secrets are used for clone.
