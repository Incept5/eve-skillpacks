---
name: eve-agent-optimisation
description: Debug and optimise Eve agents for smooth, fast execution. Covers diagnostics, prompt tuning, harness selection, team dispatch, resource management, cost control, and common failure modes across all agent workloads.
---

# Eve Agent Optimisation

Diagnose why Eve agents are slow, failing, or producing poor results — then fix them. Applies to any agent workload: coding, review, orchestration, document processing, chat routing, deployments, data analysis, or custom workflows.

## The Execution Pipeline

Every agent job follows this pipeline. Know the stages to know where to look:

```
Job created → Orchestrator claims → Route to runtime →
  Clone repo → Install skills → Hydrate resources → Build prompt →
    Select harness → Invoke eve-agent-cli →
      Agent executes (reads files, calls tools, uses CLI, emits messages) →
    Extract result → Emit events → Complete job
```

Agent jobs route to **agent-runtime** (warm pods). Build/deploy/script jobs route to **worker**. If an agent job lands on the wrong runtime, check `EVE_AGENT_RUNTIME_URL` in orchestrator config.

## Diagnostic Workflow

### Step 1: Triage

```bash
eve job diagnose <job-id>          # Full timeline, routing, errors
eve job show <job-id> --verbose    # Phase, attempts, harness, agent
```

Key signals:
- **Phase `failed`** — check `failure_disposition` (`retry` vs `permanent`).
- **Multiple attempts** — agent crashed/timed out and was retried.
- **Phase stuck `active`** — agent is running, hung, or the runtime lost it.
- **Long provisioning** — gap between `claimed` and `active` means slow clone/hydration.

### Step 2: Stream Logs

```bash
eve job follow <job-id>            # Real-time SSE log stream
```

Watch for:
- **Silence** — harness failed to start. Check runner logs.
- **Repeated tool failures** — agent trying something that won't work.
- **Excessive file reads** — agent reading too much context (bloated token usage).
- **Eve-message blocks** — agent's own progress updates to the user.

### Step 3: Inspect Results and Cost

```bash
eve job result <job-id> --format text    # Exit code + output
eve job receipt <job-id>                 # Token usage + cost breakdown
```

Compare cost against expected budget. High input tokens = agent reading too much. High output tokens = verbose output or unnecessary reasoning.

### Step 4: Check the Full Chain

```bash
eve job tree <job-id>              # Orchestrated job hierarchy
eve job dep list <job-id>          # What's blocking this job
eve job context <job-id>           # Derived status (blocked, waiting)
```

For team dispatches, check the coordination thread:
```bash
eve thread messages <thread-id> --since 1h
```

## Optimising Agent Configuration

### agents.yaml

The agent definition controls harness selection, policies, skills, gateway visibility, and API access.

```yaml
agents:
  my-agent:
    slug: my-agent
    skill: my-skill                    # primary skill loaded for this agent
    harness_profile: primary-coder     # named profile from x-eve.agents.profiles
    policies:
      permission_policy: yolo          # auto_edit | never | yolo
      git:
        commit: auto
        push: on_success
    with_apis:
      - service: api                   # app APIs the agent can call
    gateway:
      policy: routable                 # none | discoverable | routable
```

**Optimisation levers:**
- **`harness_profile`** — use profiles instead of hardcoding a harness. Profiles define a fallback chain; if the primary harness is unavailable, the next one is used.
- **`permission_policy`** — `yolo` for automated batch work (fastest). `auto_edit` for supervised coding. `default` for interactive/sensitive work.
- **`with_apis`** — only list APIs the agent actually calls. Each adds an instruction block to the prompt.
- **`gateway.policy`** — `none` for internal-only agents (hides from chat). `routable` for chat-addressable agents.
- **`skill`** — reference the specific skill, not a generic one. Skill content is the single biggest lever for agent quality.

### Harness Profiles

Define in manifest under `x-eve.agents.profiles`:

```yaml
x-eve:
  agents:
    profiles:
      fast-worker:
        - harness: mclaude
          model: haiku-4.5
          reasoning_effort: low
      deep-reviewer:
        - harness: mclaude
          model: opus-4-6
          reasoning_effort: high
        - harness: zai
          model: glm-5
          reasoning_effort: high
```

Match profile to task complexity:

| Task | Model | Reasoning | Why |
|------|-------|-----------|-----|
| Extraction, formatting, simple edits | haiku-4.5 | low | Fast, cheap, sufficient |
| General coding, analysis, reviews | sonnet-4.6 | medium | Balance of speed and quality |
| Architecture, complex debugging | opus-4-6 | high | Deep reasoning capability |
| Novel problems, multi-step proofs | opus-4-6 | x-high | Maximum thinking budget |

Reasoning effort controls thinking token budget:
- `low` ~1K tokens, `medium` ~8K, `high` ~16K, `x-high` ~32K.

### Git Controls

For code-producing agents, tune git behaviour:

```yaml
policies:
  git:
    ref: main                    # base ref to checkout
    branch: job/${job_id}        # feature branch per job
    create_branch: if_missing
    commit: auto                 # auto-commit changes on completion
    push: on_success             # push only when commits were created
```

- `commit: auto` + `push: on_success` → fastest CI loop for automated coding.
- `commit: manual` → agent decides when to commit (better for interactive work).
- `commit: required` → fail if the agent didn't produce any changes.

## Optimising Skills (SKILL.md)

The SKILL.md is the single most impactful lever for agent performance. A well-written skill eliminates wasted tool calls, prevents dead-end approaches, and guides the agent to the fastest path.

### Principles

1. **Lead with what the agent CAN do**, not prohibitions. "Read PDFs directly" beats "Do NOT use pdftotext."

2. **Be specific about inputs and outputs.** Tell the agent exactly what data it will receive (env vars, resource index, coordination inbox, app APIs) and what it should produce (attachments, control signals, commits, thread messages).

3. **Specify the output format.** JSON schema, markdown structure, or attachment name. Ambiguity wastes tokens on formatting decisions.

4. **Reference Eve primitives by name.** If the agent should use org docs, say so. If it should emit events, show the command. If it should post to a coordination thread, explain the protocol.

5. **Match instructions to the agent's runtime context.** The agent has:
   - `EVE_JOB_ID`, `EVE_PROJECT_ID`, `EVE_ATTEMPT_ID` — identity
   - `EVE_PARENT_JOB_ID` — coordination (if child job)
   - `EVE_RESOURCE_INDEX` — path to `.eve/resources/index.json` (if resources attached)
   - `.eve/coordination-inbox.md` — sibling status (if team dispatch)
   - App CLIs on `$PATH` (if `with_apis` declared with `x-eve.cli` services)
   - Org filesystem at `.org/` — shared persistent storage
   - Secrets pre-configured — auth tools work without manual setup

6. **Structure for scanning.** Agents scan before they read. Use clear headers, short paragraphs, and put the most important instruction first in each section.

### Anti-Patterns

- **Listing every possible tool** — the agent knows its tools. Only mention tools when the choice is non-obvious.
- **Verbose motivation** — agents don't need persuasion. Short, direct instructions work better.
- **Deep conditional trees** — use lookup tables instead of nested prose.
- **Referencing tools by framework name** — say "read the file" not "use the Read tool." Harnesses map to the right tool.
- **Duplicating platform docs** — reference skills and docs by name. "See `eve-orchestration` skill for decomposition patterns."

## Optimising Team Dispatch

### Choose the Right Dispatch Mode

| Mode | Behaviour | Best For |
|------|-----------|----------|
| `fanout` | Parallel child jobs per member | Independent reviews, parallel analysis |
| `council` | All respond, merged by strategy | Consensus decisions, multi-perspective review |
| `relay` | Sequential lead → member chain | Pipeline-style processing, escalation |

### Tune Team Parameters

```yaml
teams:
  review-council:
    lead: mission-control
    members: [code-reviewer, security-auditor, perf-analyst]
    dispatch:
      mode: fanout
      max_parallel: 3           # limit concurrent children
      lead_timeout: 300         # seconds before lead times out
      member_timeout: 300       # seconds per member
      merge_strategy: majority  # how to combine results
```

- **Reduce `max_parallel`** if children compete for shared resources.
- **Increase timeouts** for complex tasks; decrease for quick checks.
- **Use `relay`** when each step depends on the previous — avoids wasted parallel work.

### Coordination Thread Best Practices

Child agents receive `.eve/coordination-inbox.md` at startup. Teach skills to:
1. Read the inbox for sibling context before starting work.
2. Post progress updates: `eve thread post <thread-id> --body '{"kind":"update","body":"Phase 1 complete"}'`
3. Ask the lead for guidance: `eve thread post <thread-id> --body '{"kind":"question","body":"Should I proceed with approach A or B?"}'`

## Optimising Resource Management

### Resource Refs (Job Inputs)

Resources attached via `resource_refs` are hydrated to `.eve/resources/` before the harness starts. The index at `.eve/resources/index.json` describes what was resolved:

```json
{
  "resources": [
    {
      "uri": "org_docs:/pm/spec.md@v3",
      "local_path": ".eve/resources/pm/spec.md",
      "mime_type": "text/markdown",
      "status": "resolved"
    }
  ]
}
```

Optimisation:
- **Only attach what the agent needs.** Every ref triggers a download during provisioning.
- **Use `required: false`** for optional context — missing optionals don't fail the job.
- **Thread `mime_type`** so the agent knows file types from the index without probing.
- **Use `mount_path`** for predictable workspace locations.

### Job Attachments (Job Outputs)

Structured data passes between jobs via attachments, not prose:

```bash
eve job attach $EVE_JOB_ID --name findings.json --content '{"issues": [...]}'
eve job attach $EVE_JOB_ID --name report.md --file ./report.md
```

Parent jobs read child attachments directly:
```bash
eve job attachment $CHILD_JOB_ID findings.json --out ./child-findings.json
```

### Org Documents (Persistent Knowledge)

Agents that learn should write to the org document store:

```bash
eve docs write --org $ORG_ID --path /agents/my-agent/learnings/pattern-x.md --file ./findings.md
eve docs search --org $ORG_ID --query "auth retry"
```

Always search before writing to avoid duplicates.

### Org Filesystem (Shared State)

Agents access `.org/` in their workspace for org-wide shared files:

```bash
ls .org/shared/                           # shared knowledge base
cat .org/agents/my-agent/state.json       # agent-specific state
```

### Agent KV Store (Operational State)

For lightweight state with TTL:

```bash
eve kv set --org $ORG_ID --agent my-agent --key "pr-123-status" --value '{"phase":"review"}' --namespace workflow --ttl 86400
```

### Choose the Right Primitive

| Need | Use |
|------|-----|
| Scratch notes during execution | Workspace files (`.eve/`) |
| Pass data to parent/children | Job attachments |
| Curated persistent knowledge | Org document store |
| Shared file tree (agents + humans) | Org filesystem (`.org/`) |
| Lightweight state with TTL | Agent KV store |
| Inter-agent communication | Coordination threads |
| Encode reusable patterns | Skills |

## Optimising Workflows

### Multi-Step Workflow Design

```yaml
workflows:
  analysis-pipeline:
    steps:
      - name: gather
        agent: { name: researcher }
      - name: analyse
        depends_on: [gather]
        agent: { name: analyst }
      - name: report
        depends_on: [analyse]
        agent: { name: writer }
```

Optimisation:
- **Parallelise independent steps.** Steps without `depends_on` run concurrently.
- **Match agent to step.** Use specialised agents with focused skills instead of one general agent.
- **Right-size the harness per step.** Gathering data → haiku. Deep analysis → opus. Formatting → haiku.
- **Use `with_apis` at step level** when only specific steps need app API access.

### Event-Driven Workflows

Wire workflows to triggers for automated execution:

```yaml
workflows:
  on-doc-upload:
    trigger:
      system:
        event: doc.ingest
    steps:
      - name: process
        agent: { name: doc-processor }
```

Optimisation:
- **Scope triggers tightly.** Broad triggers create unnecessary jobs.
- **Use `dedupe_key`** on events to prevent duplicate processing.
- **Set `failure_disposition`** so transient failures retry but permanent failures don't loop.

## Cost Optimisation

### Monitor Costs

```bash
eve job receipt <job-id>                   # Per-job cost breakdown
eve job compare <job-id-1> <job-id-2>      # Compare costs across jobs
eve analytics summary --org $ORG_ID        # Org-wide analytics
eve org spend --org $ORG_ID                # Org spend overview
eve project spend --project $PROJECT_ID    # Project spend
```

### Budget Enforcement

Set cost limits on jobs to catch runaways:

```bash
eve job create --max-cost '{"currency":"USD","amount":"0.50"}' ...
```

The harness is killed if budget is exceeded. Better to fail fast than accumulate unexpected costs.

### Cost Reduction Strategies

1. **Right-size the model.** Haiku for simple tasks. Sonnet for general work. Opus only when needed.
2. **Lower reasoning effort.** `low` for extraction/formatting. `medium` for general coding. `high` only for complex analysis.
3. **Reduce input tokens.** Trim SKILL.md verbosity. Limit file reads to what's needed. Use page ranges for large documents.
4. **Reduce output tokens.** Specify concise output formats. Avoid requesting explanations when you just need the result.
5. **Parallelise with cheaper agents.** Split work across many haiku jobs instead of one opus job.
6. **Use `failure_disposition`** to prevent expensive retry loops on permanent failures.

## Common Failure Modes

### Agent Can't Start (Provisioning Failure)

**Symptom:** Job stuck between `claimed` and `active`, or fails immediately.

**Causes:**
- Git clone failure — missing credentials or bad repo URL.
- Skill install failure — broken `skills.txt` or unreachable pack source.
- Resource hydration failure — storage credentials mismatch.
- Toolchain init timeout — large toolchain images on cold pull.

**Diagnose:**
```bash
eve job diagnose <job-id>          # Check provisioning phase
eve job runner-logs <job-id>       # Raw runner/pod logs
```

**Fix:** Verify secrets (`eve secrets list`), check repo access, ensure storage credentials match.

### Agent Produces Wrong Results

**Symptom:** Job completes but output is incorrect or off-target.

**Causes:**
- SKILL.md is vague or missing key instructions.
- Wrong harness/model for the task complexity.
- Agent lacks context — missing resource refs, no coordination inbox, no access to relevant APIs.
- Prompt text is ambiguous or too broad.

**Fix:**
- Rewrite the SKILL.md with specific instructions and output format.
- Add relevant resource refs so the agent has the data it needs.
- Use a more capable model (sonnet → opus) or higher reasoning effort.
- Narrow the job description to a focused, achievable scope.

### Agent Is Too Slow

**Symptom:** Job takes much longer than expected.

**Causes:**
- Model/reasoning over-provisioned for the task.
- Agent reading too many files (context bloat).
- Large repo clone (no shallow/sparse checkout).
- Many resource refs to download.
- `permission_policy: default` requires approval for every action.

**Fix:**
- Drop to a faster model/reasoning level.
- Instruct the agent to read only specific files.
- Configure git controls: `ref_policy: auto` for minimal clone.
- Trim resource refs to essentials.
- Use `permission_policy: yolo` for automated work.

### Agent Runs Out of Context

**Symptom:** Output becomes repetitive, loses track of earlier work, or truncates.

**Fix:**
- Reduce reasoning effort (fewer thinking tokens consumed).
- Process large files in chunks (e.g., PDF page ranges).
- Split the job into orchestrated children — each child handles a focused scope.
- Use a model with a larger context window.

### Team Jobs Don't Coordinate

**Symptom:** Child agents duplicate work or miss context from siblings.

**Causes:**
- Coordination inbox not read at startup.
- Thread messages not posted during execution.
- No `depends_on` between steps that should be sequential.

**Fix:**
- Teach the skill to read `.eve/coordination-inbox.md` first.
- Add thread update messages at key milestones.
- Wire `depends_on` for steps that need prior results.

### Control Signal Issues

**Symptom:** Parent job doesn't wait for children, or stays stuck waiting.

**Causes:**
- Missing `waits_for` relations before returning `waiting` signal.
- Returning `waiting` without any blockers (triggers tight loop).
- Failed child not cleaned up — parent waits forever.

**Fix:**
- Always add `eve job dep add` before returning `waiting`.
- Handle child failures — remove or replace failed children.
- Use `eve job dep list` to verify the dependency graph.

### Auth/Secret Failures

**Symptom:** Harness fails with auth errors or tools report `401 Unauthorized`.

**Causes:**
- Missing or expired harness API key (`ANTHROPIC_API_KEY`, `Z_AI_API_KEY`, etc.).
- Git credentials not set (`github_token` or `ssh_key` secret).
- App API token expired or missing.

**Fix:**
```bash
eve harness list                   # Check harness auth availability
eve secrets list --project <id>    # Verify project secrets
eve auth sync                      # Sync local credentials
```

### Storage/Infrastructure Failures

**Symptom:** Resource hydration fails, ingest uploads fail, or org docs fail to resolve.

**Causes:**
- MinIO/S3 credential mismatch between K8s secrets and storage backend.
- AWS SDK sends headers that MinIO doesn't support (flexible checksums).
- Bucket doesn't exist or CORS not configured.

**Fix:** Verify storage credentials in K8s secrets match the actual backend. After fixing, roll out pods to pick up new secrets.

## Optimisation Checklist

Run through this when an agent isn't performing well:

- [ ] **Check the result** — exit code, token usage, cost, duration.
- [ ] **Stream logs** — identify where time is spent, what fails.
- [ ] **Review the SKILL.md** — clear, specific, output format defined.
- [ ] **Check model/reasoning** — right-sized for task complexity.
- [ ] **Check permission policy** — `yolo` for batch, `default` for interactive.
- [ ] **Check resources** — only needed refs attached, all resolved.
- [ ] **Check git controls** — branch/commit/push configured correctly.
- [ ] **Check team config** — dispatch mode, timeouts, coordination.
- [ ] **Check cost** — compare receipts against budget expectations.
- [ ] **Check routing** — agent jobs going to agent-runtime, not worker.
- [ ] **Check secrets** — harness auth available, API keys valid.
- [ ] **Check workflow design** — steps parallelised, agents specialised.

## Related Skills

- `eve-job-debugging` — CLI commands for monitoring and diagnosing jobs.
- `eve-orchestration` — decomposing work into parallel children.
- `eve-agent-memory` — storage primitives for persistence across jobs.
- `eve-skill-distillation` — encoding learned patterns into reusable skills.
- `eve-read-eve-docs` — platform reference docs (CLI, manifest, jobs, harnesses).
