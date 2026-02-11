# Jobs Reference

## Entity Model

```
Job → JobAttempt → Session → ExecutionProcess
```

- **Job**: Logical unit of work. ID: `{slug}-{hash8}` (e.g., `myproj-a3f2dd12`).
- **JobAttempt**: Isolated execution run. Has UUID `id` + job-scoped `attempt_number` (1, 2, 3...).
- **Session**: Tracks executor within an attempt. May change on reconstruction; attempt_id stays stable.
- **ExecutionProcess**: Single harness invocation within a session.

Child jobs use `{parent}.{n}` format (e.g., `myproj-a3f2dd12.1`). Max depth: 3 levels.

## Lifecycle

**Phases:** `idea` → `backlog` → `ready` → `active` → `review` → `done` | `cancelled`

Jobs default to `ready` (immediately schedulable). Priority: 0-4 (P0 highest, default 2).

**Scheduling order:** Filter phase=ready + all deps done → sort by priority (ascending) → sort by created_at (FIFO).

## API Endpoints

### Project-Scoped

```
POST /projects/{project_id}/jobs              Create job
GET  /projects/{project_id}/jobs              List jobs
GET  /projects/{project_id}/jobs/ready        Ready/schedulable jobs
GET  /projects/{project_id}/jobs/blocked      Blocked jobs
GET  /jobs                                     List jobs (admin, cross-project)
```

### Job-Scoped

```
GET    /jobs/{job_id}                          Get job
PATCH  /jobs/{job_id}                          Update job
GET    /jobs/{job_id}/tree                     Job hierarchy
GET    /jobs/{job_id}/context                  Context + derived status
GET    /jobs/{job_id}/dependencies             List dependencies
POST   /jobs/{job_id}/dependencies             Add dependency
DELETE /jobs/{job_id}/dependencies/{related}   Remove dependency
```

### Claim, Release, Attempts

```
POST /jobs/{job_id}/claim                      Claim (creates attempt, moves to active)
POST /jobs/{job_id}/release                    Release attempt
GET  /jobs/{job_id}/attempts                   List attempts
GET  /jobs/{job_id}/attempts/{n}/logs          Attempt logs
GET  /jobs/{job_id}/attempts/{n}/stream        SSE log stream for attempt
```

### Monitoring

```
GET /jobs/{job_id}/result                      Latest or attempt-specific result
GET /jobs/{job_id}/wait                        Block until completion (SSE, default 300s)
GET /jobs/{job_id}/stream                      SSE log stream for job
```

### Review Workflow

```
POST /jobs/{job_id}/submit                     Submit for review (requires summary)
POST /jobs/{job_id}/approve                    Approve (optional comment)
POST /jobs/{job_id}/reject                     Reject (requires reason)
```

### Thread Endpoints (Coordination)

```
GET  /threads/{id}/messages?since=<iso>&limit=<n>    List messages
POST /threads/{id}/messages                          Post message
```

## CLI Quick Reference

### Create

```bash
eve job create --description "Fix the login bug"
eve job create --parent myproj-a3f2dd12 --description "Sub-task"
eve job create --description "Review" --harness mclaude --model opus-4.5 --reasoning high
eve job create --description "Fix checkout" \
  --git-ref main --git-branch job/fix-checkout \
  --git-create-branch if_missing --git-commit auto --git-push on_success
```

### List and View

```bash
eve job list --phase active
eve job list --since 1h --stuck
eve job ready                                  # Schedulable jobs
eve job blocked                                # Waiting on deps
eve job show <job-id>
eve job current                                # From EVE_JOB_ID
eve job tree <job-id>
eve job diagnose <job-id>
```

### Update and Complete

```bash
eve job update <job-id> --phase active --priority 0
eve job close <job-id> --reason "Done"
eve job cancel <job-id> --reason "No longer needed"
```

### Monitor Execution

```bash
eve job follow <job-id>                        # Stream logs (SSE)
eve job wait <job-id> --timeout 120 --json     # Block until done
eve job watch <job-id>                         # Status polling + log streaming
eve job result <job-id> --format text           # Get result
eve job result <job-id> --attempt 2 --format json
eve job runner-logs <job-id>                    # kubectl pod logs
```

`wait` exit codes: 0=success, 1=failed, 124=timeout, 125=cancelled.

### Claim/Release (Agent Use)

```bash
eve job claim <job-id> --agent my-agent --harness mclaude
eve job release <job-id> --reason "Need info"
eve job attempts <job-id>
eve job logs <job-id> --attempt 2
```

### Review

```bash
eve job submit <job-id> --summary "Implemented fix, added tests"
eve job approve <job-id> --comment "LGTM"
eve job reject <job-id> --reason "Missing tests"
```

### Dependencies

```bash
eve job dep add <job-id> <depends-on-id>
eve job dep remove <job-id> <depends-on-id>
eve job dep list <job-id>
```

### Supervision and Thread Coordination

```bash
eve supervise                                  # Long-poll child events (current job)
eve supervise <job-id> --timeout 60
eve thread messages <thread-id> --since 5m
eve thread post <thread-id> --body '{"kind":"directive","body":"focus on auth"}'
eve thread follow <thread-id>
```

## Dependency Model

Relations between jobs: `blocked_by`, `blocks`, `waits_for`, `conditional_blocks`.

- `blocked_by[]`: Job IDs that must complete before this job starts.
- `blocks[]`: Sets the reverse relationship on blocking jobs.
- Scheduler filters out blocked jobs from the ready queue.

```bash
eve job create --description "Deploy to staging" # then:
eve job dep add <deploy-job> <build-job>
```

## Job Context

**Endpoint:** `GET /jobs/{job_id}/context` | **CLI:** `eve job current [--json|--tree]`

Response shape:

```
{ job, parent, children, relations: { dependencies, dependents, blocking },
  latest_attempt, latest_rejection_reason, blocked, waiting, effective_phase }
```

**Derived fields:**
- `blocked`: true when unresolved blocking relations exist.
- `waiting`: true when latest attempt returned `result_json.eve.status == "waiting"`.
- `effective_phase`: priority order `blocked` → `waiting` → `job.phase`.

Use `effective_phase` for display and orchestration decisions, not raw `phase`.

## Control Signals

Harnesses emit a `json-result` block. The worker extracts the **last** one and stores it as `job_attempts.result_json`.

```json-result
{
  "eve": {
    "status": "waiting",
    "summary": "Spawned 3 child jobs, added waits_for relations",
    "reason": "Waiting on child jobs to complete"
  }
}
```

**`eve.status` values:**
- `success`: Normal success path (review or done based on job settings).
- `waiting`: Attempt succeeds, job requeued to `ready`, assignee cleared. No review submission. If no blockers exist, orchestrator applies `defer_until` backoff to prevent tight loops.
- `failed`: Normal failure path.

**`eve.summary`**: Persisted to `job_attempts.result_summary` for quick visibility.

## Git Controls

Job-level git configuration governs ref resolution, branch creation, commit, and push behavior.

### Configuration Object

```json
{
  "git": {
    "ref": "main",
    "ref_policy": "auto",
    "branch": "job/${job_id}",
    "create_branch": "if_missing",
    "commit": "auto",
    "commit_message": "job/${job_id}: ${summary}",
    "push": "on_success",
    "remote": "origin"
  },
  "workspace": {
    "mode": "job",
    "key": "session:${session_id}"
  }
}
```

**Precedence:** explicit job fields → `x-eve.defaults.git` (manifest) → project defaults.

### Ref Resolution (`ref_policy`)

| Policy | Behavior |
|--------|----------|
| `auto` | env release SHA → manifest defaults → project default branch |
| `env` | Requires `env_name` + current release SHA |
| `project_default` | Always uses `project.branch` |
| `explicit` | Requires `git.ref` to be set |

### Repo Auth

- HTTPS: uses `github_token` secret (e.g., `GITHUB_TOKEN`).
- SSH: uses `ssh_key` secret via `GIT_SSH_COMMAND`.
- Missing auth fails fast with remediation hints (`eve secrets set`).

### Branch Creation (`create_branch`)

| Value | Behavior |
|-------|----------|
| `never` | Branch must already exist |
| `if_missing` | Create only when missing (default when `branch` is set) |
| `always` | Reset branch to `ref` |

### Commit Policy (`commit`)

| Value | Behavior |
|-------|----------|
| `never` | No commits |
| `manual` | Agent decides when to commit (default) |
| `auto` | Worker runs `git add -A` + commit after execution, even on failure |
| `required` | On success, fail attempt if working tree is clean |

### Push Policy (`push`)

| Value | Behavior |
|-------|----------|
| `never` | No push (default) |
| `on_success` | Push only when worker created commits in this attempt |
| `required` | Attempt push; no-op if no commits. Fail if push fails. |

Push without git credentials fails fast.

### Attempt Git Metadata (Audit)

Resolved values stored on attempt for debugging:

```json
{
  "resolved_ref": "refs/heads/main",
  "resolved_sha": "abc123",
  "resolved_branch": "job/myproj-a3f2dd12",
  "ref_source": "env_release|manifest|project_default|explicit",
  "pushed": true,
  "commits": ["def456"]
}
```

Also promoted to `JobResponse.resolved_git` from the latest successful attempt.

## Harness Selection

Target a harness directly or via a project profile (`x-eve.agents`):

| Flag | Purpose |
|------|---------|
| `--harness` | Harness name (mclaude, codex, gemini, zai) |
| `--profile` | Profile from `x-eve.agents` |
| `--variant` | Config overlay preset |
| `--model` | Model override |
| `--reasoning` | Effort: low, medium, high, x-high |

## Scheduling Hints

Preferences (not requirements) that influence scheduling:

| Hint | Description |
|------|-------------|
| `worker_type` | e.g., `default`, `gpu` |
| `permission_policy` | `yolo` (default), `auto_edit`, `never` |
| `timeout_seconds` | Execution timeout |

## Coordination Threads

Team dispatches create coordination threads with key `coord:job:{parent_job_id}`. Thread ID stored in `hints.coordination.thread_id`.

Child agents receive `EVE_PARENT_JOB_ID` to derive the coordination key. On attempt completion, the orchestrator auto-posts a status summary to the thread.

**Inbox file:** `.eve/coordination-inbox.md` is regenerated from recent thread messages at job start.

**Message kinds:** `status` (auto summary), `directive` (lead→member), `question` (member→lead), `update` (progress).

## Agent Environment Variables

Injected by the worker during execution:

- `EVE_PROJECT_ID` — current project
- `EVE_JOB_ID` — current job
- `EVE_ATTEMPT_ID` — current attempt UUID
- `EVE_AGENT_ID` — agent identifier
- `EVE_PARENT_JOB_ID` — parent job (for coordination)

## Not Yet Implemented

- Workspace reuse (`workspace.mode=job|session|isolated`). Today every attempt gets a fresh workspace.
- Disk LRU/TTL cleanup policies.
- Review semantics that compute diffs for branch-based jobs.
