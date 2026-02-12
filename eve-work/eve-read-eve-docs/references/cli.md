# CLI Command Reference

Complete reference for the Eve Horizon CLI (`eve`). Every command supports `--json` for machine-readable output unless noted otherwise.

## Environment Setup

Default to **staging** for user guidance. Use local/docker only when explicitly asked.

```bash
export EVE_API_URL=https://api.eh1.incept5.dev   # staging (default)
export EVE_API_URL=http://localhost:4801          # local/docker (opt-in)
export EVE_API_URL=http://api.eve.lvh.me          # k8s ingress (local)
```

Use `./bin/eh status` to discover the correct URL for the current instance.

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

## Auth

```bash
# Login / logout
eve auth login --email you@example.com                  # SSH-key login (default)
  [--ttl 30]                                            # Token TTL in days (1-90)
  [--ssh-key ~/.ssh/id_ed25519]                         # Explicit key path
  [--password]                                          # Supabase password login
  [--supabase-url <url>]                                # Custom Supabase endpoint
eve auth logout                                         # Clear local credentials

# Identity
eve auth status                                         # Show current auth state
eve auth whoami                                         # Alias for status
eve auth permissions                                    # List effective permissions
eve auth token                                          # Print current access token (raw)

# Bootstrap (first-time platform setup)
eve auth bootstrap --email you@example.com --token $EVE_BOOTSTRAP_TOKEN
  [--ssh-key <path>] [--display-name "Name"]
eve auth bootstrap --status                             # Check bootstrap eligibility

# Self-service onboarding
eve auth request-access --org "My Company" --email you@example.com
  [--ssh-key <path>] [--nostr-pubkey <hex>]
  [--wait]                                              # Poll until approved
eve auth request-access --status <request_id>           # Check request status

# Admin token minting
eve auth mint --email user@example.com --ttl 7          # Mint token for another user
  [--org <id>] [--project <id>] [--role admin]

# AI tool credential check
eve auth creds                                          # Show Claude + Codex creds
eve auth creds --claude                                 # Only check Claude
eve auth creds --codex                                  # Only check Codex
eve auth creds --json                                   # Machine-readable

# OAuth token sync to Eve
eve auth sync                                           # Sync to user-level (default)
eve auth sync --org org_xxx                             # Sync to org-level
eve auth sync --project proj_xxx                        # Sync to project-level
eve auth sync --dry-run                                 # Preview without syncing
eve auth sync --claude                                  # Only sync Claude tokens
eve auth sync --codex                                   # Only sync Codex tokens

# Service accounts (machine identity)
eve auth create-service-account --name "pm-backend" --org org_xxx \
  --scopes "jobs:create,jobs:read,projects:read"
eve auth list-service-accounts --org org_xxx
eve auth revoke-service-account --name pm-backend --org org_xxx
```

Notes:
- SSH is the default CLI login method. CLI auto-fetches SSH keys from GitHub when none are registered.
- Nostr auth uses NIP-98 request headers or kind-22242 challenge-response.
- `auth token` outputs the raw Bearer token for scripting and curl.
- `auth mint` is admin-only; creates tokens scoped to specific orgs/projects/roles.
- Service accounts create machine identities with scoped tokens for app backends.

## Access (Roles, Bindings, Policy-as-Code)

```bash
# Permission queries (require org admin)
eve access can --org org_xxx --user user_abc --permission chat:write
eve access can --org org_xxx --service-principal sp_xxx --permission jobs:read
eve access explain --org org_xxx --user user_abc --permission jobs:admin
  [--project proj_xxx]                                  # Trace permission origin

# Custom roles (additive overlays on base membership roles)
eve access roles create pm_manager --org org_xxx --scope org \
  --permissions jobs:read,jobs:write,threads:read,chat:write
eve access roles list --org org_xxx
eve access roles show pm_manager --org org_xxx
eve access roles update pm_manager --org org_xxx --add-permissions events:read
eve access roles delete pm_manager --org org_xxx

# Role bindings
eve access bind --org org_xxx --user user_abc --role pm_manager
  [--project proj_xxx]
eve access bindings list --org org_xxx [--project proj_xxx]
eve access unbind --org org_xxx --user user_abc --role pm_manager
  [--project proj_xxx]

# Policy-as-code (declarative .eve/access.yaml)
eve access validate --file .eve/access.yaml             # Validate syntax
eve access plan --file .eve/access.yaml --org org_xxx   # Preview changes
  [--json]
eve access sync --file .eve/access.yaml --org org_xxx   # Apply changes
  [--yes] [--prune]                                     # --prune removes undeclared
```

Notes:
- `can`/`explain` work for both users and service principals.
- Custom roles are additive -- they layer permissions on top of base membership roles.
- `--prune` removes roles/bindings present in API but absent from the YAML file.

## Init

```bash
eve init [directory] [--template <url>] [--branch <branch>] [--skip-skills]
```

Initialize an Eve project. Clones from template if provided, strips `.git`, runs `eve skills install` from `skills.txt` unless `--skip-skills` is set.

## Org

```bash
eve org list [--include-deleted] [--name <filter>]
eve org ensure "my-org" --slug myorg                    # Create or return existing
eve org get <org_id>                                    # Show org details
eve org update <org_id>                                 # Modify org
  [--name "New Name"] [--deleted true/false]
  [--default-agent mission-control]
  [--billing-config '{"..."}']
eve org delete <org_id>                                 # Soft-delete org
eve org spend <org_id>                                  # View org spend
  [--since 2026-01-01] [--until 2026-02-01] [--currency usd]

# Membership
eve org members --org org_xxx                           # List members
eve org members add user@example.com --role admin --org org_xxx
eve org members remove user_abc --org org_xxx
```

**URL impact:** The org `--slug` directly forms deployment URLs and K8s namespaces:
- URL: `{service}.{orgSlug}-{projectSlug}-{env}.{domain}`
- Namespace: `eve-{orgSlug}-{projectSlug}-{env}`
- `${ORG_SLUG}` is available for interpolation in manifest values (see `references/manifest.md`)

Slugs are immutable after creation. Choose short, meaningful values.

## Project

```bash
eve project list [--all] [--include-deleted] [--name <filter>]
eve project ensure --name "My Project" --slug myproj \
  --repo-url https://github.com/org/repo --branch main
  [--org org_xxx] [--force]
eve project get <project_id>
eve project update <project_id>
  [--name "New Name"] [--repo-url <url>] [--branch <branch>]
  [--deleted true/false]
eve project show <project_id>                           # Alias for get
eve project sync [--dir <path>]                         # Sync manifest to API
  [--validate-secrets] [--strict] [--project <id>]
eve project spend <project_id>                          # View project spend
  [--since 2026-01-01] [--until 2026-02-01]
  [--currency usd] [--limit 100]

# Membership
eve project members --project proj_xxx
eve project members add user@example.com --role admin --project proj_xxx
eve project members remove user_abc --project proj_xxx

# Bootstrap (create project + environments in one call)
eve project bootstrap --name my-app --repo-url https://github.com/org/repo \
  --environments staging,production [--branch main] [--slug <slug>] \
  [--template <name>] [--packs pack1,pack2]
```

Notes:
- `repo_url` accepts HTTPS, SSH (`git@host:org/repo.git`), or `file://` (local/dev only).
- `project sync` reads the manifest from `--dir` (or cwd) and pushes it to the API.

## Docs (Org Documents)

Org docs are versioned, queryable documents stored at canonical paths.

```bash
eve docs write --org org_xxx --path /pm/features/FEAT-123.md --stdin
eve docs read --org org_xxx --path /pm/features/FEAT-123.md
eve docs read --org org_xxx --path /pm/features/FEAT-123.md --version 3
eve docs show --org org_xxx --path /pm/features/FEAT-123.md --verbose
eve docs versions --org org_xxx --path /pm/features/FEAT-123.md
eve docs query --org org_xxx --path-prefix /pm/features/ \
  --where 'metadata.feature_status in draft,review' --sort updated_at:desc
eve docs search --org org_xxx --query "risk score"
eve docs delete --org org_xxx --path /pm/features/FEAT-123.md
```

## Resources (Resolver)

Resource URIs unify org docs and job attachments.

```bash
eve resources resolve org_docs:/pm/features/FEAT-123.md
eve resources resolve org_docs:/pm/features/FEAT-123.md@v4 --json
eve resources ls org_docs:/pm/features/
eve resources cat job_attachments:/myproj-a3f2dd12/plan.md
```

## Jobs

See `references/jobs.md` for lifecycle details. Jobs are the fundamental unit of work.

```bash
# Create
eve job create --description "Fix the login bug"
  [--project <id>] [--parent <id>]                     # Root or child job
  [--title "Short title"]                               # Auto-derived from description if omitted
  [--type task] [--priority 2] [--phase ready]
  [--review none|human|agent]
  [--labels "bug,urgent"] [--assignee user_abc]
  [--defer-until 2026-03-01] [--due-at 2026-03-15]
  [--env staging] [--execution-mode persistent|ephemeral]
  [--resource-refs '<json-array>']                      # Resource refs (org docs/attachments)
  [--with-apis <names>]                                 # Comma-separated API names to inject
  [--claim] [--agent <id>]                              # Create and immediately claim

  # Scheduling hints
  [--harness mclaude:fast] [--profile <name>]
  [--variant <v>] [--model <m>] [--reasoning low|medium|high|x-high]
  [--worker-type default] [--permission auto_edit]
  [--timeout 3600] [--resource-class job.c1]
  [--max-tokens 100000] [--max-cost 5.00]

  # Git controls (override project/manifest defaults)
  [--git-ref main] [--git-ref-policy auto|env|project_default|explicit]
  [--git-branch feature/fix] [--git-create-branch never|if_missing|always]
  [--git-commit never|manual|auto|required]
  [--git-commit-message "template"]
  [--git-push never|on_success|required] [--git-remote origin]

  # Workspace
  [--workspace-mode job|session|isolated]
  [--workspace-key <key>]

# List and filter
eve job list [--project <id>] [--phase ready|active|done|blocked]
  [--assignee <id>] [--priority <n>] [--since 1h]
  [--stuck] [--stuck-minutes 30]
  [--limit 50] [--offset 0]
eve job list --all [--org <id>] [--project <id>]        # Admin: cross-project listing
eve job ready [--project <id>] [--limit 10]             # Schedulable jobs shortcut
eve job blocked [--project <id>]                        # Dependency-blocked jobs

# Inspect
eve job show <job-id> [--verbose]                       # Job details (+attempts if verbose)
eve job current [<job-id>]                              # Context view (defaults to $EVE_JOB_ID)
  [--tree]                                              # Show full job tree
eve job tree <job-id>                                   # Job hierarchy tree
eve job diagnose <job-id>                               # Full diagnostic dump

# Lifecycle
eve job update <job-id> [--title] [--priority] [--phase] [--labels] ...
eve job close <job-id> [--reason "completed"]           # Close job
eve job cancel <job-id> [--reason "no longer needed"]   # Cancel job

# Dependencies
eve job dep list <job-id>                               # List dependencies
eve job dep add <job-id> <depends-on-id>                # Add dependency
eve job dep remove <job-id> <depends-on-id>             # Remove dependency

# Execution
eve job claim <job-id> [--agent <id>] [--harness <name>]  # Claim for execution
eve job release <job-id>                                # Release claim
eve job submit <job-id> [--status succeeded|failed]     # Submit result
  [--summary "Done"] [--result-json '{}']
eve job approve <job-id>                                # Approve reviewed job
eve job reject <job-id> [--reason "needs changes"]      # Reject reviewed job

# Monitoring
eve job follow <job-id>                                 # Stream harness logs (SSE)
eve job wait <job-id> [--timeout 300]                   # Block until job completes
eve job watch <job-id>                                  # Combined status + log stream
eve job runner-logs <job-id>                            # K8s runner pod logs

# Results
eve job result <job-id> [--attempt <n>]                 # Get attempt result
eve job attempts <job-id>                               # List all attempts
eve job logs <job-id> [--attempt <n>] [--after <cursor>]  # Fetch harness logs
eve job receipt <job-id>                                # Cost/token receipt
eve job compare <job-id>                                # Compare attempt results

# Attachments
eve job attach <job-id> --file <path> --name <name> [--mime <type>]
eve job attach <job-id> --stdin --name <name> [--mime <type>]
eve job attachments <job-id>                            # List attachments
eve job attachment <job-id> <name>                      # Get attachment content

# Batch operations
eve job batch --project <id> --file <path>              # Submit batch job graph
eve job batch-validate --file <path>                    # Validate batch without submitting
```

Notes:
- `--claim` on create is the inline-execution pattern: create + claim in one call.
- `--since` accepts relative time (`1h`, `30m`, `2d`) or ISO timestamps.
- `--stuck` filters to jobs active longer than expected (default or `--stuck-minutes`).
- `follow` uses SSE streaming; `watch` combines status polling with log tailing.
- `runner-logs` fetches K8s pod logs, useful for debugging harness startup failures.

## Builds

Builds are first-class primitives tracking container image construction. See `references/builds-releases.md`.

```bash
eve build list [--project <id>]                         # List build specs
eve build show <build_id>                               # Build spec details
eve build create --project <id> --ref <sha>             # Create a build spec
  --manifest-hash <hash>
  [--services <list>] [--repo-dir <path>]
eve build run <build_id>                                # Start a build run
eve build runs <build_id>                               # List runs for a build
eve build logs <build_id> [--run <id>]                  # View build logs
eve build artifacts <build_id>                          # List image artifacts (digests)
eve build diagnose <build_id>                           # Full diagnostic dump
eve build cancel <build_id>                             # Cancel active build
```

Builds happen automatically in pipeline `build` steps. Use `eve build diagnose` to debug failures.

## Releases

```bash
eve release resolve <tag> [--project <id>]              # Resolve release by tag
```

Releases are created automatically by pipeline `release` steps. See `references/builds-releases.md`.

## Pipelines

Pipeline run statuses: `pending`, `running`, `succeeded`, `failed`, `cancelled`, `awaiting_approval`.
Step types: `build`, `release`, `deploy`, `run`.

```bash
eve pipeline list [--project <id>]
eve pipeline show <project> <name>
eve pipeline run <name> --ref <sha-or-branch>
  [--env staging] [--inputs '{"k":"v"}']
  [--repo-dir ./my-app]
eve pipeline runs [project] [--status <status>]
eve pipeline show-run <pipeline> <run-id>
eve pipeline approve <run-id>
eve pipeline cancel <run-id>
eve pipeline logs <pipeline> <run-id> --step <name>
```

## Workflows

```bash
eve workflow list [--project <id>]
eve workflow show <project> <name>
eve workflow run [project] <name> --input '{"k":"v"}'   # Start async, return job-id
eve workflow invoke [project] <name> --input '{"k":"v"}'  # Start and poll for result
  [--no-wait]                                           # Return immediately
eve workflow logs <job-id> [--attempt <n>] [--after <cursor>]
```

## Environments (Deployments)

```bash
# Deploy
eve env deploy <env-name> --ref <sha-or-branch>
  [--repo-dir ./my-app]                                 # Resolve ref from local repo
  [--direct]                                            # Bypass pipeline
  [--inputs '{"k":"v"}']
  [--image-tag <tag>]                                   # Use specific image tag
  [--watch] [--timeout 300]                             # Wait for deployment to complete

# Create / manage
eve env create <name> --type persistent|temporary
  [--namespace <ns>] [--db-ref <ref>]
eve env list [project]
eve env show <project> <env>
eve env services <project> <env>                        # List running services
eve env health <project> <env>                          # Health check
eve env diagnose <project> <env> [--events]             # Full diagnostic + K8s events

# Logs
eve env logs <project> <env>
  [--since 5m] [--tail 100] [--grep "error"]
  [--pod <name>] [--container <name>]
  [--previous] [--all-pods]

# Lifecycle
eve env suspend <project> <env> --reason "maintenance"  # Pause environment
eve env resume <project> <env>                          # Resume suspended env
eve env delete <project> <env> [--force]                # Destroy environment
```

Notes:
- If a pipeline is configured, `eve env deploy` triggers that pipeline. Use `--direct` to bypass.
- `--ref` must be a 40-character SHA, or a ref resolved against `--repo-dir`/cwd.
- `env create --type temporary` creates ephemeral environments for preview/testing.
- `env suspend/resume` allow pausing environments without destroying them.

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
```

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

```bash
eve db schema --env <name> [--project <id>]             # Show DB schema
eve db rls --env <name>                                 # Show RLS policies
eve db sql --env <name> --sql "SELECT 1"                # Run query (read-only default)
  [--params '["arg1"]']                                 # Parameterized query
  [--write]                                             # Enable mutations
  [--file ./query.sql]                                  # Run SQL from file
eve db migrate --env <name> [--path db/migrations]      # Apply pending migrations
eve db migrations --env <name>                          # List applied migrations
eve db new <description> [--path db/migrations]         # Create migration file
eve db status --env <name>                              # Managed DB status
eve db rotate-credentials --env <name>                  # Rotate managed DB credentials
eve db scale --env <name> --class db.p1|db.p2|db.p3     # Scale managed DB class
eve db destroy --env <name> --force                     # Destroy managed DB
```

Migration files: `YYYYMMDDHHmmss_description.sql` in `db/migrations/` by default.

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
  [--json <payload|@file>]                              # JSON request body
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
- `--json` and `--graphql` accept inline JSON/text or `@file` paths.
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
eve analytics jobs --org org_xxx [--window 7d]           # Job analytics
eve analytics pipelines --org org_xxx [--window 7d]      # Pipeline analytics
eve analytics health --org org_xxx                       # Environment health overview
```

## Webhooks

```bash
eve webhooks create --org org_xxx --url <url> --events <evt1,evt2> --secret <secret>
  [--filter '{"key":"val"}'] [--project <id>]
eve webhooks list --org org_xxx
eve webhooks show <webhook_id> --org org_xxx
eve webhooks delete <webhook_id> --org org_xxx
eve webhooks replay <delivery_id> --org org_xxx          # Replay a delivery
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
```

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

## Debugging (CLI-first)

See `references/deploy-debug.md` for the debugging ladder and system health workflows.

Quick reference:
- `eve job diagnose <id>` -- primary job debugging entry point
- `eve job follow <id>` -- stream harness logs in real time
- `eve job runner-logs <id>` -- K8s pod logs for startup failures
- `eve env diagnose <project> <env>` -- environment health + K8s events
- `eve system health` -- platform-wide health check

## All Commands Summary

| Category | Key Commands |
|----------|-------------|
| **Profile** | `list`, `show`, `use`, `create`, `set`, `remove` |
| **Auth** | `login`, `logout`, `status`, `token`, `permissions`, `bootstrap`, `mint`, `creds`, `sync`, `request-access`, `create-service-account`, `list-service-accounts`, `revoke-service-account` |
| **Access** | `can`, `explain`, `roles create/list/show/update/delete`, `bind`, `unbind`, `bindings list`, `validate`, `plan`, `sync` |
| **Org** | `list`, `ensure`, `get`, `update`, `delete`, `spend`, `members` |
| **Project** | `list`, `ensure`, `get`, `update`, `show`, `sync`, `spend`, `members`, `bootstrap` |
| **Jobs** | `create`, `list`, `ready`, `blocked`, `show`, `current`, `tree`, `diagnose`, `update`, `close`, `cancel`, `dep`, `claim`, `release`, `attempts`, `logs`, `submit`, `approve`, `reject`, `result`, `receipt`, `compare`, `follow`, `wait`, `watch`, `runner-logs`, `attach`, `attachments`, `attachment`, `batch`, `batch-validate` |
| **Builds** | `create`, `list`, `show`, `run`, `runs`, `logs`, `artifacts`, `diagnose`, `cancel` |
| **Releases** | `resolve` |
| **Pipelines** | `list`, `show`, `run`, `runs`, `show-run`, `approve`, `cancel`, `logs` |
| **Workflows** | `list`, `show`, `run`, `invoke`, `logs` |
| **Environments** | `create`, `deploy`, `list`, `show`, `services`, `health`, `diagnose`, `logs`, `suspend`, `resume`, `delete` |
| **Secrets** | `list`, `show`, `set`, `delete`, `import`, `validate`, `ensure`, `export` |
| **Agents** | `sync`, `config`, `runtime-status` |
| **Teams** | `list` |
| **Threads** | `create`, `list`, `show`, `messages`, `post`, `follow` |
| **Packs** | `status`, `resolve` |
| **Skills** | `install` |
| **Models** | `list` |
| **Harnesses** | `list`, `get` |
| **Database** | `schema`, `rls`, `sql`, `migrate`, `migrations`, `new`, `status`, `rotate-credentials`, `scale`, `destroy` |
| **Manifest** | `validate` |
| **Providers** | `list`, `discover` |
| **Analytics** | `summary`, `jobs`, `pipelines`, `health` |
| **Webhooks** | `create`, `list`, `show`, `delete`, `replay` |
| **API** | `list`, `show`, `spec`, `refresh`, `examples`, `call`, `generate`, `diff` |
| **Events** | `list`, `show`, `emit` |
| **Chat** | `simulate` |
| **Integrations** | `list`, `slack connect`, `test` |
| **Supervision** | `supervise` |
| **Migrate** | `skills-to-packs` |
| **Admin** | `invite`, `access-requests`, `balance`, `usage`, `pricing`, `receipts`, `models` |
| **System** | `status`, `health`, `config`, `settings`, `orchestrator`, `jobs`, `envs`, `logs`, `pods`, `events` |
