# Jobs (Current)

## Lifecycle + Phases

Phases: `idea` -> `backlog` -> `ready` -> `active` -> `review` -> `done` (or `cancelled`).
Jobs default to `ready`.

Priority: 0-4 (P0 highest).

## Job IDs

- Root: `{slug}-{hash8}` (e.g., `myproj-a3f2dd12`)
- Child: `{parent}.{n}` (e.g., `myproj-a3f2dd12.1`)

## Core CLI

```bash
eve job create --description "Fix bug"
eve job list --phase active
eve job show <job-id>
eve job update <job-id> --phase backlog

eve job follow <job-id>
eve job logs <job-id> --attempt 2
eve job diagnose <job-id>

eve job submit <job-id> --summary "Done"
eve job approve <job-id>
eve job reject <job-id> --reason "Missing tests"
```

### Dependencies

```bash
eve job dep add <job-id> <depends-on-id>
eve job dep list <job-id>
```

### Attempts + Claim

```bash
eve job claim <job-id> --agent my-agent --harness mclaude
eve job release <job-id> --reason "Need info"
eve job attempts <job-id>
```

## Harness Selection (Per Job)

```bash
eve job create --description "Review" --harness mclaude --model opus-4.5 --reasoning high
# or use a profile from x-eve.agents

eve job create --description "Review" --profile primary-reviewer --reasoning x-high
```

Relevant fields:
- `harness` (mclaude/codex/gemini/zai)
- `harness_profile` (from `x-eve.agents`)
- `harness_options` (variant/model/reasoning)
- `hints` (worker_type, permission_policy, timeout)

## Job Context

`GET /jobs/:id/context` (CLI: `eve job current --json`) returns:
- job + parent + children
- relations (dependencies/dependents)
- `blocked` (has unmet deps)
- `waiting` (attempt returned `eve.status = "waiting"`)
- `effective_phase` (blocked -> waiting -> phase)

## Git Controls (Current)

Job-level git controls govern ref/branch/commit/push behavior.

```json
{
  "git": {
    "ref": "main",
    "ref_policy": "auto|env|project_default|explicit",
    "branch": "job/${job_id}",
    "create_branch": "never|if_missing|always",
    "commit": "never|manual|auto|required",
    "commit_message": "job/${job_id}: ${summary}",
    "push": "never|on_success|required",
    "remote": "origin"
  },
  "workspace": {
    "mode": "job|session|isolated",
    "key": "session:${session_id}"
  }
}
```

Defaults are taken from `x-eve.defaults.git` and `x-eve.defaults.workspace`.

Key behaviors:
- `ref_policy=auto`: env release -> manifest defaults -> project branch.
- `commit=auto`: worker runs `git add -A` and commits any changes.
- `commit=required`: fail if no commit on success.
- `push=on_success`: push only if worker created commits.

Workspace reuse (`workspace.mode`) is **not yet enforced**; each attempt gets a fresh workspace today.

## Scheduling Hints (Current)

Hints influence scheduling but do not replace harness settings:
- `worker_type`
- `permission_policy`
- `timeout_seconds`

## Planned (Not Implemented)

- Workspace reuse across job/session.
- Disk LRU/TTL cleanup policies.
- Review semantics that compute diffs for branch-based jobs.
