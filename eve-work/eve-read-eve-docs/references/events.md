# Events + Triggers (Current)

## Event Model

Events are stored in Postgres and routed by the orchestrator.
Core fields: `type`, `source`, `status`, `payload_json`, `env_name`, `ref_sha`, `ref_branch`, `actor_type`, `actor_id`, `dedupe_key`.

Sources: `github`, `slack`, `cron`, `manual`, `app`, `system`.

## System Failure Events

- `system.job.failed`
- `system.pipeline.failed`

## API + CLI

```bash
eve event list [project] --type github.push --source github
eve event show <event-id>
eve event emit --type manual.test --source manual --payload '{"k":"v"}'
```

## Trigger Routing

The orchestrator polls pending events, matches manifest triggers, and creates pipeline runs or workflow jobs.

Example trigger:

```yaml
trigger:
  github:
    event: pull_request
    action: [opened, synchronize]
    base_branch: main
```

Supported PR actions: `opened`, `synchronize`, `reopened`, `closed`.
Branch patterns support wildcards (e.g., `release/*`).

## Planned (Not Implemented)

- Cron scheduling and event emission from manifest.
- Stronger dedupe/idempotency semantics.
