# CLI Command Reference

## Use When
- You need exact Eve CLI commands, options, and output behavior.
- You need to establish CLI context for org/project/environment or auth state.
- You need cross-resource workflows (jobs, pipelines, manifests, docs, secrets) from one interface.

## Load Next
- `references/overview.md` for platform orientation and environment setup.
- `references/jobs.md`, `references/pipelines-workflows.md`, or `references/manifest.md` for task-specific flags.
- `references/secrets-auth.md` for auth/scope-sensitive workflows.

## Ask If Missing
- Confirm whether the user is on staging, docker, or local k3d and the correct `EVE_API_URL`.
- Confirm target org/project IDs or slugs before crafting commands.
- Confirm any required auth mode (`eve auth login`, bootstrap, or service principal) for the action.

Complete reference for the Eve Horizon CLI (`eve`). Every command supports `--json` for machine-readable output unless noted otherwise.

**Data envelope:** All list endpoints return `{ "data": [...] }` in JSON mode. The CLI handles both wrapped and unwrapped responses transparently, so agents should always expect the `data` wrapper when parsing `--json` output from list commands.

## Environment Setup

Default to **staging** for user guidance. Use local/docker only when explicitly asked.

```bash
export EVE_API_URL=https://api.eh1.incept5.dev   # staging (default)
export EVE_API_URL=http://localhost:4801          # local/docker (opt-in)
export EVE_API_URL=http://api.eve.lvh.me          # k8s ingress (local k3d stack)
```

Managed Ollama targets live on the same API host using `/inference/v1`:

```bash
export EVE_MANAGED_OLLAMA_URL="${EVE_API_URL%/}/inference/v1"
```

Examples:

```bash
# staging
export EVE_MANAGED_OLLAMA_URL=https://api.eh1.incept5.dev/inference/v1

# local/docker
export EVE_MANAGED_OLLAMA_URL=http://localhost:4801/inference/v1
```

Use `./bin/eh status` to discover the correct URL for the current instance. For local development, `eve local up` provisions a full k3d stack and prints the correct API URL.

## Profile

Profiles are **repo-local** configuration bundles. They store API URL, default org/project, harness preference, and auth identity.

```bash
eve profile list                                        # List all profiles
eve profile show [name]                                 # Show profile details (default: active)
eve profile use <name> [--clear]                        # Switch active profile
eve profile create <name> --api-url <url>               # Create a new profile
  [--org <id>] [--project <id>] [--harness <name>]
  [--default-email <email>] [--default-ssh-key <path>]
  [--supabase-url <url>] [--supabase-anon-key <key>]
eve profile set [name] --org <id>                       # Update profile fields
  [--project <id>] [--api-url <url>] [--harness <name>]
  [--default-email <email>] [--default-ssh-key <path>]
eve profile remove <name>                               # Delete a profile
```

Notes:
- Profiles persist to `.eve/profiles.json` in the repo root.
- `--clear` on `use` resets all fields to defaults.
- Set `--org` and `--project` to avoid passing them on every command.

## Task Modules

Load the module that matches the current question before opening full command blocks:

- `references/cli-auth.md` -- login, bootstrap, service accounts, roles, bindings, policy-as-code.
- `references/cli-org-project.md` -- init, org/project lifecycle, docs, FS sync, resource resolver.
- `references/cli-jobs.md` -- job create/list/show/join/follow, attachments, and batch workflows.
- `references/cli-pipelines.md` -- builds, releases, pipelines, and workflow commands.
- `references/cli-deploy-debug.md` -- environment deploy/recover/rollback, local k3d stack, and CLI-first debugging.

For any topic, open only the matching module to keep load minimal.

## Secrets

Secrets support four scopes: `--project`, `--org`, `--user`, `--system`. Project scope is the default.

```bash
eve secrets list --project proj_xxx
eve secrets show <key> --project proj_xxx               # Show secret metadata
eve secrets set <key> <value> --project proj_xxx
eve secrets delete <key> --project proj_xxx
eve secrets import --org org_xxx --file ./secrets.env    # Bulk import from .env file
eve secrets validate --project proj_xxx                 # Check manifest-required secrets
  [--keys KEY1,KEY2]
eve secrets ensure --project proj_xxx --keys KEY1,KEY2  # Ensure keys exist
eve secrets export --project proj_xxx --keys KEY1       # Export values
```

## Agents, Teams + Chat

See `references/agents-teams.md` for full agent and team configuration details.

```bash
# Sync config from repo to API
eve agents sync --project proj_xxx --ref <sha>
eve agents sync --project proj_xxx --local --allow-dirty
  [--force-nonlocal]                                    # Force non-local sync

# View effective config (pack resolution pipeline)
eve agents config [--repo-dir <path>] [--no-harnesses] [--json]

# Agent runtime status
eve agents runtime-status --org org_xxx [--json]
```

## Teams

```bash
eve teams list [--json]
```

## Threads

```bash
# Org-scoped thread management
eve thread create --org org_xxx --key <key>             # Create org thread
eve thread list --org org_xxx [--scope <scope>]         # List org threads
  [--key-prefix <prefix>]
eve thread show <thread-id> --org org_xxx               # Show thread details

eve thread messages <thread-id>                         # List messages
  [--since 5m] [--limit 20] [--json]
eve thread post <thread-id>                             # Post a message
  --body '{"kind":"update","body":"text"}'
  [--actor-type user] [--actor-id <id>] [--job-id <id>]
eve thread follow <thread-id>                           # Tail messages (polls every 3s)

# Distillation (summarize thread into memory or docs)
eve thread distill <thread-id> --org org_xxx
  [--to <doc-path>]                                     # Write distillation to org docs path
  [--agent <slug>] [--category <name>] [--key <key>]    # Write to agent memory
  [--prompt "custom distillation prompt"]
  [--auto --threshold <n> --interval <duration>]        # Auto-distill on message threshold
```

Notes:
- `distill` summarizes a thread's messages into a condensed form. Output can be routed to an org doc path (`--to`) or to agent memory (`--agent` + `--category` + `--key`).
- `--auto` enables automatic re-distillation when message count exceeds `--threshold` since last distillation, checked on `--interval`.

## KV (Agent Key-Value Store)

Namespaced key-value storage scoped to an agent within an org. Values can be any JSON type.

```bash
eve kv set --org org_xxx --agent <slug> --key <key> --value <json-or-string>
  [--namespace <ns>] [--ttl <seconds>]                  # Optional namespace (default: "default") and TTL
eve kv get --org org_xxx --agent <slug> --key <key>
  [--namespace <ns>]
eve kv list --org org_xxx --agent <slug>                # List keys in namespace
  [--namespace <ns>] [--limit <n>]
eve kv mget --org org_xxx --agent <slug> --keys a,b,c   # Batch get multiple keys
  [--namespace <ns>]
eve kv delete --org org_xxx --agent <slug> --key <key>
  [--namespace <ns>]
```

Notes:
- Namespace defaults to `"default"` when omitted.
- `--value` is parsed as JSON if valid, otherwise stored as a string.
- `--ttl` sets a time-to-live in seconds; expired keys are not returned.

## Memory (Agent Memory)

Structured long-term memory for agents. Entries are organized by agent, category, and key, with support for lifecycle metadata, tagging, confidence scores, and cross-agent search.

```bash
eve memory set --org org_xxx (--agent <slug>|--shared)
  --category <category> --key <key>
  (--file <path>|--stdin|--content <text>)              # Content source
  [--tags "architecture,decision"]                      # Comma-separated tags
  [--confidence 0.9]                                    # Confidence score (0-1)
  [--supersedes <memory-id>]                            # Supersede a previous entry
  [--lifecycle-status active|archived|draft]
  [--review-in 30d] [--expires-in 90d]                  # Lifecycle scheduling

eve memory get --org org_xxx (--agent <slug>|--shared)
  --key <key> [--category <category>]

eve memory list --org org_xxx (--agent <slug>|--shared)
  [--category <category>] [--tags a,b] [--limit <n>]

eve memory delete --org org_xxx (--agent <slug>|--shared)
  --category <category> --key <key>

eve memory search --org org_xxx --query <text>           # Cross-agent memory search
  [--agent <slug>] [--limit <n>]
```

Notes:
- `--shared` stores memory in the org-wide shared namespace (agent slug `shared`).
- `--supersedes` links the new entry to a previous one, forming a revision chain.
- `search` queries across all agents in the org unless `--agent` scopes it to one.
- Lifecycle fields (`--review-in`, `--expires-in`) work identically to org docs.

## Search (Unified Cross-Source Search)

Federated search across all org data sources: memory, docs, threads, attachments, and events.

```bash
eve search --org org_xxx --query <text>
  [--sources memory,docs,threads,attachments,events]    # Comma-separated source filter
  [--agent <slug>]                                      # Scope to agent's data
  [--limit <n>]
```

Notes:
- When `--sources` is omitted, all sources are searched.
- Results are ranked by relevance and grouped by source type.

## Packs (AgentPacks)

```bash
eve packs status [--repo-dir <path>]                    # Show lockfile status + drift
eve packs resolve [--dry-run] [--repo-dir <path>]       # Preview pack resolution
```

## Skills

```bash
eve skills install [source] [--skip-installed]          # Install skill packs
```

- Without source: reads `skills.txt` and installs all entries.
- With source: installs directly from URL, GitHub repo, or local path and persists to `skills.txt`.
- Supports: `https://github.com/org/repo`, `org/repo` (GitHub shorthand), `./local/path`.
- Installs for all supported agents: claude-code, codex, gemini-cli.

## Models + Harnesses

```bash
# Models
eve models list [--managed] [--json]                    # List available LLM models

# Harnesses
eve harness list [--capabilities] [--org <id>] [--project <id>]
eve harness get <name> [--org <id>] [--project <id>]
```

Notes:
- `--managed` filters to platform-managed models (vs. BYOK).
- `--capabilities` shows model support, reasoning levels, streaming, and tool use.
- `harness get` shows variants, auth status, and capability matrix.

## Database (Environment DBs)

All `db` subcommands accept either `--env <name>` (API-proxied) or `--url <postgres-url>` (direct connection). When `--url` is used, the CLI connects directly to the database without going through the Eve API. The `EVE_DB_URL` env var (or `.env` file) is used as a fallback for `sql` and `migrate`.

```bash
eve db schema --env <name> [--project <id>]             # Show DB schema
eve db schema --url <postgres-url>                      # Show schema via direct connection
eve db rls --env <name>                                 # Show RLS policies + group context diagnostics
eve db rls init --with-groups [--out <path>] [--force]  # Scaffold group-aware RLS helper SQL
eve db sql --env <name> --sql "SELECT 1"                # Run query (read-only default)
  [--params '["arg1"]']                                 # Parameterized query
  [--write]                                             # Enable mutations
  [--file ./query.sql]                                  # Run SQL from file
eve db sql --url <postgres-url> --sql "SELECT 1"        # Direct SQL execution
eve db migrate --env <name> [--path db/migrations]      # Apply pending migrations
eve db migrate --url <postgres-url> [--path db/migrations]  # Direct migration apply
eve db migrations --env <name>                          # List applied migrations
eve db new <description> [--path db/migrations]         # Create migration file
eve db reset --env <name> --force [--no-migrate]        # Drop and recreate schema, re-apply migrations
  [--danger-reset-production] [--path db/migrations]
eve db reset --url <postgres-url> --force               # Direct schema reset
eve db wipe --env <name> --force                        # Reset schema without re-applying migrations
eve db status --env <name>                              # Managed DB status
eve db rotate-credentials --env <name>                  # Rotate managed DB credentials
eve db scale --env <name> --class db.p1|db.p2|db.p3     # Scale managed DB class
eve db destroy --env <name> --force                     # Destroy managed DB
```

Notes:
- `rls` now includes group context diagnostics: shows the resolved principal, org, project, env, group IDs, and permissions for the current session. Useful for debugging why policies are not matching.
- `rls init --with-groups` scaffolds `app.current_user_id()`, `app.current_group_ids()`, and `app.has_group()` SQL functions to `db/rls/helpers.sql` (or `--out <path>`). Apply the output SQL to your target environment DB, then reference these helpers in RLS policies.
- Migration files: `YYYYMMDDHHmmss_description.sql` in `db/migrations/` by default.
- `reset` drops all schemas (except `pg_catalog`, `information_schema`) and re-applies migrations. Use `--no-migrate` (or `wipe`) to skip migration re-apply.
- `--danger-reset-production` is required when resetting production environments via the API.
- `--url` mode uses the `@eve/migrate` library for direct `migrate`, `migrations`, `reset`, and `wipe` operations.

## Manifest

```bash
eve manifest validate [--path <path>]                   # Validate manifest
  [--validate-secrets] [--strict] [--latest]
  [--project <id>] [--dir <path>]
```

See `references/manifest.md` for manifest schema and configuration.

## API (Proxy)

Call project API sources (OpenAPI, PostgREST, Supabase GraphQL) through Eve's auth layer.

```bash
eve api list [project] [--env <name>]                   # List API sources
eve api show <name> [project] [--env <name>]            # Show API source details
eve api spec <name> [project] [--env <name>]            # Show cached API spec
eve api refresh <name> [project] [--env <name>]         # Refresh cached spec
eve api examples <name> [project] [--env <name>]        # Generate curl examples from spec

eve api call <name> <method> <path>                     # Call API with Eve auth
  [--json <payload|@file>] [--data <payload|@file>]     # JSON request body
  [--graphql <query|@file>]                             # GraphQL query
  [--variables '{"k":"v"}']                             # GraphQL variables
  [--jq <expr>]                                         # Filter response with jq
  [--print-curl]                                        # Print curl command instead
  [--token <override>]                                  # Custom auth token
  [--project <id>] [--env <name>]
```

eve api generate [--out <dir>]                          # Export OpenAPI spec
eve api diff [--exit-code] [--out <dir>]                # Diff generated vs committed spec
```

Notes:
- `call` resolves the base URL from the API source, applies Eve auth, and proxies the request.
- `--json` and `--data` are aliases (`-d` shorthand also works); use only one body flag.
- `--json`/`--data` and `--graphql` accept inline JSON/text or `@file` paths.
- `--env` falls back to `$EVE_ENV_NAME` when omitted.
- `call` rewrites service DNS/ingress base URLs to match runtime context (local shell vs in-cluster job).
- `--jq` requires `jq` installed locally.
- Auth precedence: `--token` > `$EVE_JOB_TOKEN` > profile token.
- `generate` exports the current OpenAPI spec. `diff` compares against committed spec.

## Chat + Integrations (Slack, Nostr)

```bash
# Integrations
eve integrations list --org org_xxx
eve integrations slack connect --org org_xxx --team-id T123
  --token xoxb-test [--tokens-json '{}'] [--status]
eve integrations test <integration_id> --org org_xxx

# Default agent slug (org-wide fallback)
eve org update org_xxx --default-agent mission-control

# Chat simulation (project-scoped)
eve chat simulate --project proj_xxx
  --team-id T123 --channel-id C123 --user-id U123 --text "hello"
  [--provider slack] [--thread-key <key>] [--metadata '{}']
```

Slack commands (run inside Slack):

```text
@eve <agent-slug> <command>
@eve agents list
@eve agents listen <agent-slug>
@eve agents unlisten <agent-slug>
@eve agents listening
```

Nostr relay subscriptions provide a non-webhook transport. See `references/gateways.md`.

## Events (Triggers)

```bash
eve event list --project proj_xxx
  [--type github.push] [--source github] [--status <s>]
eve event show <event-id>
eve event emit --type manual.test --source manual
  [--env staging] [--ref-sha <sha>] [--ref-branch main]
  [--actor-type user] [--actor-id <id>]
  [--dedupe-key <key>]
  --payload '{"k":"v"}'
```

See `references/events.md` for the complete event type catalog and trigger syntax.

## Supervision

```bash
eve supervise [<job-id>] [--timeout 60] [--since <cursor>] [--json]
```

Monitors job tree and team coordination. Returns events, children, inbox, and cursor for long-polling.

## Migrate

```bash
eve migrate skills-to-packs                             # Convert skills.txt to x-eve.packs YAML
```

Reads `skills.txt` and generates the equivalent `x-eve.packs` manifest fragment for migration from legacy skills to the AgentPacks system.

## Providers

```bash
eve providers list [--json]                             # List registered providers
eve providers discover <provider> [--json]              # Discover models for a provider
```

Notes:
- Providers are first-class entities with auth config, harness mapping, and model discovery.
- `discover` fetches live model lists from the provider's API (cached with TTL).

## Analytics

```bash
eve analytics summary --org org_xxx [--window 7d]       # Org-wide summary
eve analytics jobs --org org_xxx [--window 7d]           # Job counters (created/completed/failed/active)
eve analytics pipelines --org org_xxx [--window 7d]      # Pipeline success rates and durations
eve analytics env-health --org org_xxx                   # Environment health snapshot (total/healthy/degraded/unknown)
```

Notes:
- All analytics endpoints return aggregate counters, not per-item listings. Use `--json` for machine-readable output.
- `--window` accepts relative durations like `7d`, `24h`, `30d`.

## Webhooks

```bash
eve webhooks create --org org_xxx --url <url> --events <evt1,evt2> --secret <secret>
  [--filter '{"key":"val"}'] [--project <id>]
eve webhooks list --org org_xxx
eve webhooks show <webhook_id> --org org_xxx
eve webhooks deliveries <webhook_id> --org org_xxx [--limit 50]
eve webhooks test <webhook_id> --org org_xxx
eve webhooks delete <webhook_id> --org org_xxx
eve webhooks enable <webhook_id> --org org_xxx
eve webhooks replay <webhook_id> --org org_xxx
  [--from-event <event_id>] [--to <iso-time>] [--max-events <n>] [--dry-run]
eve webhooks replay-status <webhook_id> <replay_id> --org org_xxx
```

Notes:
- Webhook subscriptions support HMAC signature verification and retry logic.
- Filters scope events to specific patterns. Project-scoped webhooks are optional.

## Admin

Administrative commands (require elevated permissions).

```bash
# User invitation
eve admin invite --email user@example.com --github user
  [--web] [--redirect-to <url>]                         # --web sends Supabase invite email

# Access request management
eve admin access-requests list
eve admin access-requests approve <request_id>
eve admin access-requests reject <request_id> --reason "..."

# Billing
eve admin balance show <org_id>                         # Current balance
eve admin balance credit <org_id> --amount 100.00       # Add credit
  [--currency usd] [--description "reason"]
eve admin balance transactions <org_id>                 # Transaction history
  [--limit 50] [--offset 0]

eve admin usage list --org org_xxx                      # Usage records
  [--since 2026-01-01] [--until 2026-02-01] [--limit 100]
eve admin usage summary --org org_xxx                   # Aggregated usage summary
  [--since 2026-01-01] [--until 2026-02-01]

# Pricing
eve admin pricing seed-defaults                         # Seed default rate cards

# Receipts
eve admin receipts recompute                            # Recompute cost receipts
  [--since 2026-01-01] [--project proj_xxx]
  [--dry-run] [--force]

# Model management (org/project scoped)
eve admin models list --org org_xxx [--project proj_xxx]
eve admin models set <name> --org org_xxx               # Set managed model
  [--project proj_xxx] [--harness <harness>] [--model <model>]
eve admin models delete <name> --org org_xxx [--project proj_xxx]

# Ingress aliases (custom domain aliases for services)
eve admin ingress-aliases list                          # List all aliases
  [--alias <alias>] [--project <id>] [--environment <id>]
  [--limit <n>] [--offset <n>]
eve admin ingress-aliases reclaim <alias> --reason "..."  # Reclaim an alias
```

Notes:
- Access-request approval is retry-safe (`approve` returns the existing approved record on repeat calls).
- Duplicate identity fingerprints are attached to the existing identity owner.
- `ingress-aliases reclaim` forcibly releases an alias from its current project/environment. Requires a `--reason`.

## System (Internal)

```bash
eve system status                                       # Platform status overview
eve system health [--json]                              # Health check
eve system config                                       # Show system configuration
eve system settings                                     # List all settings
eve system settings get <key>                           # Get setting value
eve system settings set <key> <value>                   # Set setting value

eve system orchestrator status                          # Orchestrator state
eve system orchestrator set-concurrency <n>             # Set job concurrency

eve system jobs [--org <id>] [--project <id>]           # Active jobs overview
  [--phase <p>] [--limit 50]
eve system envs [--org <id>] [--project <id>]           # Environment overview
eve system logs <service> [--tail 50]                   # Service logs
eve system pods                                         # K8s pod status
eve system events [--limit 50]                          # Recent platform events
```

## Ollama (Inference Targets)

```bash
eve ollama targets [--scope-kind <platform|org|project>] [--scope-id <id>]
eve ollama target add --name <name> --base-url <url>
  [--scope-kind <kind>] [--scope-id <id>]
  [--target-type external_ollama|internal_pool]
  [--transport-profile ollama_api|openai_compat]
  [--api-key-ref <secret_ref>]
eve ollama target rm <target_id>
eve ollama target test <target_id> [--org-id <id>] [--project-id <id>]
eve ollama target wake <target_id> [--wait=true|false] [--timeout-ms <ms>]

eve ollama models
eve ollama model add --canonical <model_id> --provider <provider> --slug <provider_model_slug>

eve ollama installs [--target-id <id>] [--model-id <id>]
eve ollama install add --target-id <id> --model-id <id>
  [--requires-warm-start true|false] [--min-target-capacity <n>]
eve ollama install rm --target-id <id> --model-id <id>

eve ollama aliases [--scope-kind <kind>] [--scope-id <id>]
eve ollama alias set --alias <name> --target-id <target_id> --model-id <model_id>
  [--scope-kind <kind>] [--scope-id <id>]
eve ollama alias rm --alias <name> [--scope-kind <kind>] [--scope-id <id>]

eve ollama assignments [--scope-kind <kind>] [--scope-id <id>]
eve ollama route-policies [--scope-kind <kind>] [--scope-id <id>]
eve ollama route-policy set --scope-kind <kind> [--scope-id <id>]
  --preferred-target-id <target_id>
  [--fallback-to-alias-target true|false]
eve ollama route-policy rm --scope-kind <kind> [--scope-id <id>]
```

Notes:
- Use installs + route policy to switch between local and external Ollama endpoints.
- `fallback-to-alias-target=true` keeps requests flowing if preferred target is not eligible for the model.
- Managed inference traffic is served from `.../inference/v1` on the active API URL (for example, `https://api.eh1.incept5.dev/inference/v1` in staging), matching the OpenAPI route `/inference/v1/chat/completions`.
- Inference requests also accept `OLLAMA_API_KEY` or `x-ollama-api-key` headers as Bearer-token-compatible alternatives to `Authorization: Bearer`.
- `/inference/v1/chat/completions` usage gates are controlled by:
  - `inference:write` (primary),
  - `jobs:write` (compatibility fallback)
  - `EVE_INFERENCE_ORG_TOKENS_PER_HOUR`
  - `EVE_INFERENCE_PROJECT_TOKENS_PER_HOUR`
- `eve ollama target wake` is used for scale-to-zero target pools and supports `--wait` plus `--timeout-ms`.

## Local Stack (k3d)

Provision and manage a full Eve platform stack locally using k3d (k3s-in-Docker). Requires Docker Desktop. The CLI auto-installs `k3d` and `kubectl` into `~/.eve/bin/` if not already present.

Supported platforms: macOS and Linux (amd64 and arm64).

```bash
# Bring up the local stack (creates cluster, pulls images, migrates DB, starts all services)
eve local up
  [--version <x.y.z>]                                  # Platform version (default: latest)
  [--skip-deploy]                                      # Start cluster only, skip image deploy
  [--skip-health]                                      # Don't wait for API health check
  [--timeout <seconds>]                                # Rollout timeout (default: 300)
  [--verbose]                                          # Show kubectl/docker output
  [--json]                                             # Machine-readable output

# Stop the local cluster (preserves state)
eve local down
  [--destroy]                                          # Delete cluster and all data
  [--force]                                            # Skip confirmation prompt

# Full status dashboard
eve local status
  [--watch]                                            # Auto-refresh every 5s
  [--json]

# Health check (exits non-zero if unhealthy)
eve local health [--json]

# Destroy and recreate from scratch
eve local reset
  [--force]                                            # Skip confirmation prompt

# Stream service logs
eve local logs [<service>]                             # Omit service for all logs
  [--follow] [-f]                                      # Tail logs
  [--tail <n>]                                         # Lines to show (default: 50)
  [--since <duration>]                                 # e.g. 5m, 1h
```

**Services:** api, orchestrator, worker, gateway, agent-runtime, auth, mailpit, sso, postgres.

**Local URLs** (available after `eve local up`):
- API: `http://api.eve.lvh.me`
- Auth: `http://auth.eve.lvh.me`
- Mail: `http://mail.eve.lvh.me`
- SSO: `http://sso.eve.lvh.me`

**Typical workflow:**
```bash
eve local up                                           # First run: ~5min (image pulls)
export EVE_API_URL=http://api.eve.lvh.me
eve org ensure "my-org" --slug my-org                  # Bootstrap an org
eve local status --watch                               # Monitor services
eve local logs api --follow                            # Debug API issues
eve local down                                         # Stop (state preserved)
eve local up                                           # Restart (fast, no re-pull)
eve local reset --force                                # Nuclear option: destroy + recreate
```

Notes:
- `up` is idempotent: re-running starts a stopped cluster or skips if already running.
- `down` without `--destroy` preserves cluster state; `up` resumes quickly.
- `reset` is equivalent to `down --destroy` followed by `up`.
- Version resolution queries the configured registry for the latest common platform tag.
  - Set `ECR_REGISTRY=<registry>` to override the registry host.
  - Set `ECR_NAMESPACE=<namespace>` to override the image namespace (defaults to `eve-horizon`).
- The cluster binds ports 80/443 on localhost via k3d's load balancer.
- Kube context is set to `k3d-eve-local` automatically.

## Debugging (CLI-first)

See `references/deploy-debug.md` for the debugging ladder and system health workflows.

Quick reference:
- `eve job diagnose <id>` -- primary job debugging entry point
- `eve job follow <id>` -- stream harness logs in real time
- `eve job runner-logs <id>` -- K8s pod logs for startup failures
- `eve env diagnose <project> <env>` -- environment health + K8s events
- `eve env recover <project> <env>` -- analyze state and suggest recovery action
- `eve system health` -- platform-wide health check

## All Commands Summary

| Category | Key Commands |
|----------|-------------|
| **Profile** | `list`, `show`, `use`, `create`, `set`, `remove` |
| **Auth** | `login`, `logout`, `status`, `token`, `permissions`, `bootstrap`, `mint`, `creds`, `sync`, `request-access`, `create-service-account`, `list-service-accounts`, `revoke-service-account` |
| **Access** | `can`, `explain`, `roles create/list/show/update/delete`, `bind`, `unbind`, `bindings list`, `groups create/list/show/update/delete`, `groups members list/add/remove`, `memberships`, `validate`, `plan`, `sync` |
| **Org** | `list`, `ensure`, `get`, `update`, `delete`, `spend`, `members` |
| **Project** | `list`, `ensure`, `get`, `update`, `show`, `sync`, `spend`, `members`, `bootstrap`, `status` |
| **Docs** | `write`/`create`, `read`, `show`, `list`, `search`, `stale`, `review`, `versions`, `query`, `delete` |
| **Jobs** | `create`, `list`, `ready`, `blocked`, `show`, `current`, `tree`, `diagnose`, `update`, `close`, `cancel`, `dep`, `claim`, `release`, `attempts`, `logs`, `submit`, `approve`, `reject`, `result`, `receipt`, `compare`, `follow`, `wait`, `watch`, `runner-logs`, `attach`, `attachments`, `attachment`, `batch`, `batch-validate` |
| **Builds** | `create`, `list`, `show`, `run`, `runs`, `logs`, `artifacts`, `diagnose`, `cancel` |
| **Releases** | `resolve` |
| **Pipelines** | `list`, `show`, `run`, `runs`, `show-run`, `approve`, `cancel`, `logs` |
| **Workflows** | `list`, `show`, `run`, `invoke`, `logs` |
| **Environments** | `create`, `deploy`, `list`, `show`, `services`, `health`, `diagnose`, `logs`, `rollback`, `reset`, `recover`, `suspend`, `resume`, `delete` |
| **FS Sync** | `init`, `status`, `logs`, `pause`, `resume`, `disconnect`, `mode`, `conflicts`, `resolve`, `doctor` |
| **Secrets** | `list`, `show`, `set`, `delete`, `import`, `validate`, `ensure`, `export` |
| **Agents** | `sync`, `config`, `runtime-status` |
| **Teams** | `list` |
| **Threads** | `create`, `list`, `show`, `messages`, `post`, `follow`, `distill` |
| **KV** | `set`, `get`, `list`, `mget`, `delete` |
| **Memory** | `set`, `get`, `list`, `delete`, `search` |
| **Search** | `search` (unified cross-source) |
| **Packs** | `status`, `resolve` |
| **Skills** | `install` |
| **Models** | `list` |
| **Ollama** | `targets`, `target add/rm/test/wake`, `models`, `model add`, `installs`, `install add/rm`, `aliases`, `alias set/rm`, `assignments`, `route-policies`, `route-policy set/rm` |
| **Harnesses** | `list`, `get` |
| **Database** | `schema`, `rls`, `rls init --with-groups`, `sql`, `migrate`, `migrations`, `new`, `reset`, `wipe`, `status`, `rotate-credentials`, `scale`, `destroy` |
| **Manifest** | `validate` |
| **Providers** | `list`, `discover` |
| **Analytics** | `summary`, `jobs`, `pipelines`, `env-health` |
| **Webhooks** | `create`, `list`, `show`, `delete`, `replay` |
| **API** | `list`, `show`, `spec`, `refresh`, `examples`, `call`, `generate`, `diff` |
| **Events** | `list`, `show`, `emit` |
| **Chat** | `simulate` |
| **Integrations** | `list`, `slack connect`, `test` |
| **Supervision** | `supervise` |
| **Migrate** | `skills-to-packs` |
| **Local Stack** | `up`, `down`, `status`, `health`, `reset`, `logs` |
| **Admin** | `invite`, `access-requests`, `balance`, `usage`, `pricing`, `receipts`, `models`, `ingress-aliases` |
| **System** | `status`, `health`, `config`, `settings`, `orchestrator`, `jobs`, `envs`, `logs`, `pods`, `events` |
