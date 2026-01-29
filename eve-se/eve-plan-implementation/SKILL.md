---
name: eve-plan-implementation
description: Execute software engineering plan documents using Eve jobs, dependencies, and review gating.
---

# Eve Plan Implementation (Jobs)

Translate a plan document into Eve jobs, parallelize work, and drive review/verification
through job phases and dependencies.

## When to Use

- A plan/spec exists and the work should be orchestrated as Eve jobs.
- The initial job says "use eve-plan-implementation to implement the plan."

## Workflow

### 1) Load context

- Read the plan doc and extract phases, deliverables, and blockers.
- If present, read `AGENTS.md` for repo-specific rules.
- Fetch current job context:
  ```bash
  eve job current --json
  ```

### 2) Create or confirm the root epic

If the root job does not exist, create one:
```bash
eve job create \
  --project $EVE_PROJECT_ID \
  --description "Implement <plan name>" \
  --review human \
  --phase backlog
```

If a root job already exists, use it as the orchestrator.

### 3) Break down into phase jobs

Create one child job per plan phase (backlog or ready). Include a short scope and deliverable.

```bash
eve job create \
  --project $EVE_PROJECT_ID \
  --parent $EVE_JOB_ID \
  --description "Phase: <name>. Deliverable: <artifact/result>" \
  --phase ready
```

Add dependencies so the parent waits on each phase:
```bash
eve job dep add $EVE_JOB_ID $PHASE_JOB_ID --type waits_for
```

### 4) Create task jobs under each phase

Split each phase into 2-6 atomic tasks with clear deliverables:

```bash
eve job create \
  --project $EVE_PROJECT_ID \
  --parent $PHASE_JOB_ID \
  --description "Task: <objective>. Deliverable: <result>" \
  --phase ready
```

Make the phase wait on its tasks:
```bash
eve job dep add $PHASE_JOB_ID $TASK_JOB_ID --type waits_for
```

### 5) Parallelize by default

- Independent tasks should have no dependencies and run in parallel.
- Use `blocks` only for true sequencing requirements.

### 6) Execute tasks and update phases

For each task:
```bash
eve job update $TASK_JOB_ID --phase active
# do the work
eve job submit $TASK_JOB_ID --summary "Completed <deliverable>"
```

If no review is required:
```bash
eve job close $TASK_JOB_ID --reason "Done"
```

### 7) Verification and review

- Add a dedicated verification job (tests, manual checks) gated after
  implementation tasks.
- Submit the phase job when all tasks are complete.
- When all phases complete, submit the root epic for review.

### 8) Orchestrator waiting signal (optional)

If the root or phase job is only coordinating children, return a waiting signal
after dependencies are created:

```json-result
{
  "eve": {
    "status": "waiting",
    "summary": "Spawned child jobs and added waits_for relations"
  }
}
```

## Minimal Mapping from Beads to Eve Jobs

- Epic -> root job (issue_type via `--type` if available).
- Phase -> child job under the epic.
- Task -> child job under the phase.
- `bd dep add` -> `eve job dep add <parent> <child> --type waits_for`
- `bd ready/blocked` -> `eve job dep list <id>` + `eve job list --phase ...`

## Optional: Git controls template

If tasks require code changes on a shared branch:

```bash
eve job create \
  --project $EVE_PROJECT_ID \
  --description "Task: <objective>" \
  --git-ref main \
  --git-ref-policy explicit \
  --git-branch feature/<name> \
  --git-create-branch if_missing \
  --git-commit required \
  --git-push on_success
```

Keep git controls consistent across tasks so all changes land in one PR.
