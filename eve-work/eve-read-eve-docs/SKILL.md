---
name: eve-read-eve-docs
description: Load first. State-today index of distilled Eve Horizon system docs with task-based routing for CLI/API usage, manifests, pipelines, jobs, secrets, agents, builds, events, and debugging.
triggers:
  - eve docs
  - eve horizon docs
  - read eve docs
  - eve cli
  - eve manifest
  - eve pipelines
  - eve workflows
  - eve job
  - eve secrets
  - eve auth
  - eve events
  - eve triggers
  - eve agents
  - eve teams
  - eve builds
  - eve releases
  - eve deploy
  - eve observability
  - eve traces
  - eve env logs
  - eve env diagnose
  - eve filesystem
  - eve fs sync
  - eve object store
  - eve sdk
  - eve chat sdk
  - eve conversation sdk
  - eve embedded conversation
  - eve conversations
  - eve app conversations
  - eve auth sdk
  - eve sso
  - eve integrations
  - eve slack
  - eve github
  - eve identity
  - eve ingest
  - eve ingestion
  - eve document
  - eve pdf
  - eve media
  - eve audio
  - eve video
  - eve whisper
  - eve ffmpeg
  - eve google drive
  - eve cloud fs
  - eve cloud-fs
  - eve gcs
  - eve oauth
  - eve per-org oauth
  - eve BYOA
  - eve BYOK
  - eve endpoint
  - eve tailscale
  - eve private endpoint
  - eve toolchain
  - eve harness
  - eve harness override
  - eve harness validate
  - eve env override
  - eve job env override
  - eve job harness override
  - eve staged dispatch
  - eve agent alias
  - eve chat delivery
  - eve chat progress
  - eve event trigger
  - eve app trigger
  - eve workflow optimization
  - eve undeploy
  - eve app delete
  - eve traces query
  - eve env logs follow
  - eve env logs filter
  - eve env diagnose request
  - eve domain
  - eve custom domain
  - custom domain
  - eve stable egress
  - egress
  - hostNetwork egress
  - deploy failure
  - DeployFailure
  - deploy diagnose
  - eve sentinel
  - platform sentinel
  - env health monitoring
  - eve service token
  - EVE_SERVICE_TOKEN
  - eve signup domain
  - signup domain restriction
  - managed db tls
  - db tls trust
  - verify-full
  - eve docs patch
  - eve docs diff
  - eve docs watch
  - eve docs sync
  - eve docs list tree
  - app bucket credentials
  - eve app storage
  - eve skills materialize
  - runtime skills
  - sparse pack
  - system.job.attempt.completed
  - learning loop
  - carryover context
  - embedded conversation
  - conversation events
  - cevt
  - system app pattern
  - eve dashboard
  - step level harness
  - conditional workflow step
  - workflow env_overrides
  - pipeline env propagation
  - workflow retry failed
  - workflow file ref
  - chat case insensitive
  - org_default
  - org auto discovery
  - app cli
  - agent cli
  - cli for agents
  - cli wrapper
  - eve wiki
  - llm wiki
  - wiki pattern
  - knowledge base
  - wiki maintenance
---

# Eve Read Docs (Load First)

Purpose: provide a compact, public, always-available distillation of Eve Horizon system docs. Use this when private system docs are not accessible.

## When to Use

- Any question about how to use Eve Horizon via CLI or API.
- Any question about `.eve/manifest.yaml`, pipelines, workflows, jobs, or secrets.
- Any question about events, triggers, agents, teams, builds, or deployments.

## How to Use

1. Start with `references/overview.md` for core concepts, IDs, and the reference index.
2. Use the task router below to choose the smallest set of references for the request.
3. Open only the relevant reference files and avoid loading unrelated docs.
4. Ask for missing project or environment inputs before giving prescriptive commands.

## Task Router (Progressive Access)

- Platform orientation, environment URLs, architecture, Eve Dashboard, system app pattern: `references/overview.md`
- Command syntax, flags, and CLI workflows (includes cloud-fs, endpoint, ingest, traces, env logs --follow/--filter, env diagnose --request commands): `references/cli.md`
- Fine-grained CLI intents:
  - `references/cli-auth.md` (auth + access + policy)
  - `references/cli-org-project.md` (init, org/project setup, docs, fs sync)
  - `references/cli-jobs.md` (jobs and execution controls, per-job harness/env overrides)
  - `references/cli-pipelines.md` (builds, releases, pipelines, workflows)
  - `references/cli-deploy-debug.md` (deploy, recovery, local stack, CLI troubleshooting, env logs follow/filter, env diagnose --request, traces query)
- Manifest authoring, config structure, app CLI framework, toolchain declarations, cloud FS mounts, per-org OAuth, app undeploy/delete, custom domains, stable egress (hostNetwork v2), workflow env_overrides + conditional steps + step-level harness/harness_options + step git controls + retry tails + file refs + Slack notifications + resource ref policies, manifest-driven service token permissions: `references/manifest.md`
- Pipelines, workflows, triggers, event-driven automation, auto-trigger, event/app triggers, workflow input forwarding, step optimization, per-step `with_apis`, workflow env_overrides + conditional steps + step-level harness + retry-failed + file refs + Slack notifications, event→trigger observability (trigger_match_count, triggers_evaluated): `references/pipelines-workflows.md` + `references/events.md`
- Job lifecycle, scheduling, execution debugging, agent-native monitoring, production hardening, per-job HOME isolation, per-job harness/env overrides, learning loop (`system.job.attempt.completed`, carryover context), stuck-job prevention + stale recovery + env-gate scope: `references/jobs.md`
- Build, release, and deployment behavior: `references/builds-releases.md` + `references/deploy-debug.md`
- Private endpoints (Tailscale), worker toolchain-on-demand, app undeploy/delete, custom domains debugging (first-bind-wins, cert-manager TLS, `eve domain list|verify|status|transfer|unbind|remove`), stable egress (hostNetwork v2), DeployFailure taxonomy + cluster snapshot + manifest_hash from deploy ref + `eve env diagnose`, Platform Sentinel (env health monitoring + Slack alerts): `references/deploy-debug.md`
- Agents, teams, chat routing, embedded app conversations, agent aliases, staged dispatch, chat delivery, chat progress, structured conversation event streams (`cevt_*`), chat continuity by Eve `thr_*` id, chat regex case-insensitive, agent learning loop hooks, agent-runtime org auto-discovery (no `org_default`): `references/agents-teams.md` + `references/gateways.md`
- Secrets, auth, access control, identity providers, BYOK model credentials, per-org OAuth credential storage, manifest-driven service token permissions + auto-injected `EVE_SERVICE_TOKEN` (read-only defaults), SSO self-signup email domain restriction (`EVE_SIGNUP_ALLOWED_EMAIL_DOMAINS`), per-agent envdb wildcard scope (built-in roles): `references/secrets-auth.md`
- Skills installation, packs, resolution order, materialization fast-path + `.agents/skills/` canonicalization + sparse pack support + `eve skills materialize` runtime path: `references/skills-system.md`
- Harness selection, sandbox policy, BYOK model setup, shared invoke, toolchain-on-demand, chat harness profiles, per-job harness override (`--harness-override-file`) + env override (`--env-override`), harness-profile-validation endpoint, chat hint propagation, Phase 4 step-template expressions, Codex reasoning + harness model normalization, Opus 4.7 + GPT-5.5 model registrations: `references/harnesses.md`
- Object store, org filesystem sync, share tokens, public paths, GCS storage, cloud FS (Google Drive), app bucket credential separation: `references/object-store-filesystem.md`
- Document ingestion (upload, processing, download, callbacks): `references/ingest.md`
- Document ingestion pipeline (end-to-end flow, agentpack, media processing, chat files, doc.ingest workflow trigger reliability fixes): `references/document-ingestion.md`
- Eve SDK overview, install, quick-start, token flow, embedded conversations, chat SDKs, exports, structured conversation event streams, chat continuity by `thr_*` id: `references/eve-sdk.md`
- Auth SDK deep-dive, `@eve-horizon/auth`, `@eve-horizon/auth-react`, app SSO middleware, token verification, project role resolution, and org awareness: `references/auth-sdk.md`
- Build agent-friendly CLIs for app APIs, manifest declaration, bundling, distribution, env var contract: `references/app-cli.md`
- OAuth app credentials (BYOA), Google Drive mounts, cloud FS browse/search, Slack install smoothing, gateway hot-load, per-org OAuth, chat file materialization, integrations, Slack connect, GitHub setup, identity linking, membership requests, API chat provider (no-op for polling clients; 4 built-in providers: slack, nostr, webchat, api): `references/integrations.md` + `references/gateways.md`
- Observability, request diagnostics, service logs (`eve env logs --follow`, `--filter k=v`), traces (`eve traces query`), `eve env diagnose --request <req_id>`, cost tracking, receipts, analytics, event→trigger observability (trigger_match_count, triggers_evaluated, `eve event show`): `references/observability.md`
- Database provisioning, migrations, SQL, managed DB operations, managed DB TLS trust (verify-full default + CA bundle injection): `references/database-ops.md`
- Symptom-first troubleshooting across auth, secrets, deploy, jobs, builds, DeployFailure taxonomy + diagnose, Platform Sentinel alert interpretation, stuck-job prevention + stale recovery: `references/troubleshooting.md`
- LLM Wiki pattern, wiki-maintaining agents, knowledge base architecture, near-instant indexing, CLI enhancements (`eve docs patch`, `diff`, `watch`, `sync`, `list --tree`, `search --path/--context`, bulk-write): `references/llm-wiki.md`

## Index

- `references/overview.md` -- Architecture, core concepts, IDs, job phases, BYOK inference, document ingestion, cloud FS, private endpoints, Eve Dashboard + system app pattern, reference index.
- `references/cli.md` -- CLI quick reference: all commands by category with flags and options, including cloud-fs, endpoint, ingest, traces query, env logs --follow/--filter, env diagnose --request commands.
- `references/manifest.md` -- Manifest v2 spec: services, environments, pipelines, workflows, x-eve extensions, app CLI framework, toolchain declarations, cloud FS mounts, per-org OAuth, app undeploy/delete, custom domains, stable egress (`x-eve.networking.egress: stable`), workflow env_overrides + conditional steps + step-level harness/harness_options + step git controls + retry tails + file refs + Slack notifications + resource ref policies, manifest-driven service token permissions.
- `references/events.md` -- **Event type catalog** (all sources + payloads) and **trigger syntax** (github, slack, system, cron, manual).
- `references/jobs.md` -- Job lifecycle, phases, CLI, git/workspace controls, scheduling hints, agent-native monitoring, production hardening, per-job HOME isolation, per-job harness/env overrides, learning loop (`system.job.attempt.completed`, carryover context, `user` memory category), stuck-job prevention + stale recovery + env-gate scope (action jobs only).
- `references/builds-releases.md` -- Build system (specs, runs, artifacts), releases, deploy model, promotion patterns.
- `references/agents-teams.md` -- Agent/team/chat YAML schemas, sync flow, slug rules, embedded app conversations, agent aliases, staged team dispatch, chat outbound delivery, chat progress updates, dispatch modes, coordination threads, structured conversation event streams (`cevt_*`), chat continuity by `thr_*` id, chat regex case-insensitive, agent learning loop hooks, agent-runtime org auto-discovery.
- `references/pipelines-workflows.md` -- Pipeline steps, triggers, workflow invocation, auto-trigger, event/app triggers, workflow input forwarding, step optimization, per-step `with_apis`, build-release-deploy pattern, env_overrides + conditional steps + step-level harness + step git controls + retry-failed + file refs + Slack notifications + resource ref policies, event→trigger observability (trigger_match_count, triggers_evaluated).
- `references/secrets-auth.md` -- Secrets scopes, interpolation, auth model, identity providers, OAuth sync, service principals, access visibility, custom roles, policy-as-code, BYOK model credentials, per-org OAuth credential storage, manifest-driven service token permissions + auto-injected `EVE_SERVICE_TOKEN` (read-only defaults), SSO self-signup email domain restriction (`EVE_SIGNUP_ALLOWED_EMAIL_DOMAINS`), per-agent envdb wildcard scope.
- `references/skills-system.md` -- Skills format, skills.txt, install flow, discovery priority, materialization fast-path + `.agents/skills/` canonicalization + sparse pack support + `eve skills materialize` runtime path.
- `references/deploy-debug.md` -- K8s architecture, worker images, deploy polling, ingress/TLS, secrets provisioning, workspace janitor, private endpoints (Tailscale), worker toolchain-on-demand, app undeploy/delete, CLI debugging workflows, real-time debugging, env-specific debugging, custom domains debugging (first-bind-wins, cert-manager TLS, `eve domain` subcommands), stable egress (hostNetwork v2), DeployFailure taxonomy + cluster snapshot + manifest_hash from deploy ref + `eve env diagnose`, Platform Sentinel (env health monitoring + Slack alerts).
- `references/harnesses.md` -- Harness selection, profiles, auth priority, sandbox flags, BYOK model setup, shared invoke module, toolchain-on-demand, harness profiles for chat, per-job harness override (`--harness-override-file`) + env override (`--env-override`), harness-profile-validation endpoint, chat hint propagation, Phase 4 step-template expressions, Codex reasoning + harness model normalization, Opus 4.7 + GPT-5.5 model registrations.
- `references/gateways.md` -- Gateway plugin architecture, Slack + Nostr + webchat + API providers (4 built-in), thread keys, structured conversation event streams (`cevt_*`), chat continuity by Eve `thr_*` id, embedded app conversations, API chat provider (no-op for polling clients).
- `references/cli-auth.md` -- CLI auth, service accounts, access roles, and policy-as-code.
- `references/cli-org-project.md` -- CLI commands for org/project setup, docs, FS sync, and resolver URIs.
- `references/cli-jobs.md` -- CLI job lifecycle: create/list/update, attempt tracking, result/monitoring/attachments, per-job harness override (`--harness-override-file`) + env override (`--env-override`).
- `references/cli-pipelines.md` -- CLI build/release/pipeline/workflow command reference.
- `references/cli-deploy-debug.md` -- CLI environment deploy/recover/lifecycle and local k3d stack, env logs --follow/--filter, env diagnose --request, traces query.
- `references/object-store-filesystem.md` -- Object store, org filesystem sync protocol, share tokens, public paths, app buckets, access control, native GCS storage, cloud FS (Google Drive mounts), app bucket credential separation.
- `references/ingest.md` -- Document ingest lifecycle: upload, processing, download URLs, callbacks, CORS, event integration.
- `references/document-ingestion.md` -- Document ingestion pipeline: end-to-end flow, ingest:// URI scheme, agentpack, media processing (ffmpeg + whisper), chat file materialization, integration points, doc.ingest workflow trigger reliability fixes.
- `references/eve-sdk.md` -- Eve SDK overview: auth + chat packages, install, quick-start patterns, token flow, embedded conversation pane, backend/frontend exports, environment variables, structured conversation event streams, chat continuity by `thr_*` id.
- `references/auth-sdk.md` -- Eve Auth SDK deep-dive: middleware behavior, verification strategies, token types, SSO session bootstrap, NestJS patterns, project role resolution, org awareness, migration guide.
- `references/integrations.md` -- OAuth app credentials (BYOA), Google Drive cloud FS mounts, Slack install smoothing, gateway hot-load, per-org OAuth, chat file materialization, external integrations (Slack, GitHub), identity resolution tiers, membership requests, CLI linking.
- `references/observability.md` -- Correlation IDs, app service logs (`eve env logs --follow`, `--filter k=v`), request diagnostics (`eve env diagnose --request`), traces (`eve traces query`), execution receipts, cost tracking, analytics, OTEL config, provider discovery, event→trigger observability (trigger_match_count, triggers_evaluated, `eve event show`).
- `references/database-ops.md` -- Managed DB provisioning, migrations, SQL access, schema/RLS inspection, scaling/reset/destroy, managed DB TLS trust (verify-full default + CA bundle injection).
- `references/app-cli.md` -- App CLI framework: manifest declaration, env var contract, bundling, distribution, implementation patterns.
- `references/troubleshooting.md` -- Symptom-first diagnostic tables for auth, secrets, deploy, jobs, builds, network issues, DeployFailure taxonomy + `eve env diagnose`, Platform Sentinel alert interpretation, stuck-job prevention + stale recovery.
- `references/llm-wiki.md` -- LLM Wiki pattern: two-layer substrate, agent workflow, near-instant indexing, operations (ingest/query/lint/bulk), CLI enhancements (`eve docs patch`, `diff`, `watch`, `sync`, `list --tree`, `search --path/--context`, bulk-write), relationship to Company as Intelligence.

## Intent Coverage Matrix

| Intent | Minimum references | Expected output |
|---|---|---|
| Authenticate or inspect permissions | `references/cli-auth.md`, `references/secrets-auth.md` | Session state, token/permission validation result |
| Bootstrap org/project resources | `references/cli-org-project.md`, `references/manifest.md` | Org/project IDs, members, manifest sync status |
| Submit and monitor work | `references/cli-jobs.md`, `references/jobs.md` | Job IDs, phase transitions, attempt logs |
| Build/deploy a version | `references/cli-pipelines.md`, `references/builds-releases.md`, `references/pipelines-workflows.md` | Pipeline run ID, build/release artifacts, deployment trace |
| Recover from runtime issues | `references/cli-deploy-debug.md`, `references/deploy-debug.md`, `references/cli-jobs.md` | Diagnose output, recovery target, mitigation command plan |
| Trace a deployed app request | `references/observability.md`, `references/cli-deploy-debug.md`, `references/deploy-debug.md` | Request logs, deploy metadata, K8s events, audit rows, trace spans |
| Inspect platform behavior or events | `references/events.md`, `references/agents-teams.md` | Canonical event stream view, routing path |
| Install/update skills for agents | `references/skills-system.md`, `references/overview.md` | Installed pack/skill set and resolution order |
| Monitor costs, receipts, or analytics | `references/observability.md`, `references/cli.md` | Receipt breakdown, analytics counters, cost totals |
| Provision or operate environment databases | `references/database-ops.md`, `references/manifest.md` | Migration status, query results, managed DB state |
| Sync files, share links, or configure org filesystem | `references/object-store-filesystem.md`, `references/cli-org-project.md` | Sync status, share tokens, public path URLs |
| Upload, process, or download documents via ingest | `references/ingest.md`, `references/document-ingestion.md`, `references/events.md` | Ingest IDs, download URLs, callback payloads, processing status |
| Configure ingest agentpack or media processing | `references/document-ingestion.md`, `references/agents-teams.md` | Pack import, profile selection, media tool availability |
| Understand how Slack files reach agents | `references/document-ingestion.md`, `references/gateways.md` | Chat file flow, attachment index, workspace layout |
| Add SSO auth to an app or verify tokens | `references/eve-sdk.md`, `references/auth-sdk.md`, `references/secrets-auth.md` | SDK setup code, token verification, SSO flow |
| Add an embedded agent conversation pane | `references/eve-sdk.md`, `references/agents-teams.md`, `references/gateways.md` | Conversation API shape, SDK setup code, route policy, stream resume behavior |
| Configure OAuth app credentials or connect Google Drive | `references/integrations.md` | BYOA config status, mount IDs, browse/search results |
| Connect Slack/GitHub or resolve external identities | `references/integrations.md`, `references/agents-teams.md` | Integration status, identity binding, membership requests |
| Build an agent-friendly CLI for an app API | `references/app-cli.md`, `references/manifest.md` | CLI source, esbuild bundle, manifest declaration, tested commands |
| Connect Google Drive or browse cloud FS | `references/object-store-filesystem.md`, `references/integrations.md` | Mount ID, browse/search results, cloud FS event triggers |
| Set up private endpoints (Tailscale) | `references/deploy-debug.md`, `references/cli.md` | Endpoint name, in-cluster DNS, health check status |
| Declare or use toolchains | `references/manifest.md`, `references/harnesses.md` | Toolchain list, init container config, PATH setup |
| Set up agent aliases or vanity names | `references/agents-teams.md` | Alias binding, slug resolution, sync validation |
| Check chat delivery status or progress | `references/agents-teams.md`, `references/gateways.md` | Delivery status, thread messages, progress updates |
| Configure event or app triggers for workflows | `references/pipelines-workflows.md`, `references/events.md` | Trigger config, event payload forwarding, matched workflow |
| Undeploy or delete an app/environment | `references/deploy-debug.md`, `references/manifest.md` | Deploy status, cleanup sequence, cascade-delete |
| Configure BYOK model credentials | `references/secrets-auth.md`, `references/harnesses.md` | Secret scope, harness env mapping, provider key |
| Set up per-org OAuth credentials (BYOA) | `references/integrations.md`, `references/manifest.md` | OAuth app config, provider connection, credential storage |
| Configure GCS native storage | `references/object-store-filesystem.md` | Storage backend config, Workload Identity binding |
| Monitor jobs with agent-native tooling | `references/jobs.md`, `references/cli-jobs.md` | Job status, monitoring output, production safeguards |
| Set up staged team dispatch (council) | `references/agents-teams.md` | Team config, staged flag, lead/member coordination |
| Configure per-step `with_apis` in workflows | `references/pipelines-workflows.md`, `references/manifest.md` | Step overrides, API injection, workflow-level defaults |
| Resolve project roles in auth SDK | `references/auth-sdk.md`, `references/secrets-auth.md` | Project role claim, `X-Eve-Project-Id` header, middleware config |
| Configure custom domains for deployed apps | `references/cli.md`, `references/manifest.md`, `references/deploy-debug.md` | Domain registration, DNS verification, TLS activation |
| Bind a custom domain | `references/manifest.md`, `references/deploy-debug.md` | Domain binding, first-bind-wins behavior, cert-manager TLS status |
| Diagnose a failure from symptoms | `references/troubleshooting.md`, `references/deploy-debug.md` | Root cause, fix command, recovery path |
| Override harness or env per job | `references/cli-jobs.md`, `references/jobs.md`, `references/harnesses.md` | Override file path, env keys, validated harness profile |
| Validate a harness profile | `references/harnesses.md` | Validation result, normalized model, sandbox/auth check |
| Configure conditional or step-level harness in workflows | `references/pipelines-workflows.md`, `references/manifest.md` | Step harness/harness_options block, condition expression, env_overrides |
| Diagnose a failed deploy | `references/deploy-debug.md`, `references/troubleshooting.md` | DeployFailure code, cluster snapshot, manifest_hash, diagnose output |
| Configure stable egress (hostNetwork) | `references/manifest.md`, `references/deploy-debug.md` | `x-eve.networking.egress: stable` config, source IP, pod placement |
| Interpret Platform Sentinel alerts | `references/troubleshooting.md`, `references/deploy-debug.md` | Alert source, env health probe, Slack message, remediation |
| Stream service logs or filter on JSON keys | `references/observability.md`, `references/cli-deploy-debug.md` | Live log stream, filtered key/value matches |
| Diagnose a single request end-to-end | `references/observability.md`, `references/cli-deploy-debug.md` | Request timeline, spans, logs, K8s context for `req_*` |
| Query OTEL traces from CLI | `references/observability.md`, `references/cli.md` | Trace query results, span tree, latency breakdown |
| Continue an Eve thread by `thr_*` ID | `references/agents-teams.md`, `references/eve-sdk.md` | Thread resume call, conversation continuity, structured event stream |
| Edit, diff, or watch wiki docs from CLI | `references/llm-wiki.md` | `eve docs patch/diff/watch/sync` flow, near-instant index status |
| Restrict SSO self-signup by email domain | `references/secrets-auth.md` | `EVE_SIGNUP_ALLOWED_EMAIL_DOMAINS` config, allowlist behavior |
| Materialize skills at job runtime | `references/skills-system.md` | `eve skills materialize` invocation, `.agents/skills/` layout, sparse pack subset |
| Use the agent learning loop / job.attempt.completed | `references/jobs.md`, `references/agents-teams.md` | Event payload, carryover context, `user` memory writes |
| Adopt the system app pattern (deploy a platform-tier app) | `references/overview.md` | Pattern definition, system app boundaries, cross-link to fullstack-app-design |
| Configure manifest-driven service tokens | `references/secrets-auth.md`, `references/manifest.md` | Permission scopes, auto-injected `EVE_SERVICE_TOKEN`, read-only defaults |
| Verify managed DB TLS trust | `references/database-ops.md` | `verify-full` default, CA bundle injection path, connection string |
| Separate app bucket credentials | `references/object-store-filesystem.md` | App bucket credential scope, isolation boundary |

## Hard Rules

- Eve is **API-first**; the CLI only needs `EVE_API_URL`.
- Do **not** assume URLs, ports, or environment state--ask if unknown.
- These references describe shipped platform behavior only.
- If anything is missing or unclear, ask for the missing inputs.
