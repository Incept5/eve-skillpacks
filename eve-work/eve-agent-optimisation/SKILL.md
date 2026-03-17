---
name: eve-agent-optimisation
description: Debug and optimise Eve agents for smooth, fast execution. Covers diagnostics, resource hydration, prompt tuning, harness configuration, and common failure modes.
---

# Eve Agent Optimisation

Diagnose why Eve agents are slow, failing, or producing poor results — then fix them. This skill covers the full agent execution pipeline from job creation to harness completion.

## The Execution Pipeline

Understand the pipeline to know where to look when things go wrong:

```
Job created → Orchestrator claims → Route to runtime →
  Clone repo → Hydrate resources → Build prompt →
    Select harness → Invoke eve-agent-cli →
      Agent executes (reads files, runs tools, writes output) →
    Extract result → Emit events → Complete job
```

Every stage can be a bottleneck or failure point. Optimisation means identifying which stage is slow or broken and addressing it directly.

## Diagnostic Workflow

### Step 1: Get the Full Picture

```bash
eve job diagnose <job-id>
eve job show <job-id> --verbose
```

Check for:
- **Phase stuck at `active`** — agent is still running or hung.
- **Phase at `failed`** — check `failure_disposition` for retry vs permanent failure.
- **Multiple attempts** — agent crashed or timed out and was retried.
- **Long time between `claimed` and `active`** — provisioning bottleneck (clone, resource hydration).

### Step 2: Stream Logs in Real Time

```bash
eve job follow <job-id>
```

Watch for:
- **Long gaps between output lines** — agent may be stuck in a tool call or waiting for LLM response.
- **Repeated tool failures** — agent is trying something that won't work (missing binary, wrong path, permission denied).
- **No output at all** — harness failed to start, check runner logs.

### Step 3: Check the Result

```bash
eve job result <job-id> --format text
```

Look at:
- **Exit code** — 0 is success, non-zero indicates harness or agent failure.
- **Token usage** — high input tokens may indicate excessive file reading; high output tokens may indicate verbosity.
- **Cost** — compare against expected budget.

### Step 4: Check Resource Hydration

```bash
eve job show <job-id> --verbose
```

Examine `resource_refs` in the job data. Each ref has a URI scheme:
- `ingest://` — uploaded documents from document ingestion.
- `org_docs://` — versioned org documents.
- `job://` — attachments from prior jobs.

If resources failed to hydrate, the agent starts without its inputs. Common causes:
- Storage credentials mismatch (MinIO/S3 auth).
- Document not found (deleted or wrong version).
- Network timeout downloading large files.

## Resource Hydration Optimisation

Resources are downloaded to `.eve/resources/` in the workspace. The agent reads them via the resource index.

### The Resource Index

Located at `.eve/resources/index.json` (path exposed via `EVE_RESOURCE_INDEX` env var):

```json
{
  "resolved_at": "2026-03-17T10:00:00Z",
  "resources": [
    {
      "uri": "ingest://ing_abc123/report.pdf",
      "local_path": ".eve/resources/report.pdf",
      "content_hash": "sha256:abc123...",
      "mime_type": "application/pdf",
      "metadata": {
        "title": "Q4 Report",
        "instructions": "Extract key metrics"
      },
      "label": "report.pdf",
      "required": true,
      "status": "resolved"
    }
  ]
}
```

### Key Fields for Agent Intelligence

- **`mime_type`** — tells the agent what kind of file it's dealing with before opening it. A well-written SKILL.md uses this to choose the right processing strategy (native PDF reading vs text extraction vs image analysis).
- **`metadata`** — carries submitter context (title, description, instructions, tags). The agent should read this to understand what the user wants done with the file, not just what the file is.
- **`status`** — `resolved` means the file is available; `missing` means it wasn't found. Check `required` — missing optional resources are normal, missing required resources are failures.

### Threading Metadata Through the Pipeline

When building workflows that ingest documents, ensure the full chain preserves context:

```
CLI upload (mime_type, title, instructions) →
  Event payload (system.doc.ingest) →
    Workflow resource_refs (mime_type, metadata) →
      Resource index (mime_type, metadata) →
        Agent reads index → Knows file type + intent
```

If the agent doesn't know the file type, it wastes time probing or guessing. Ensure every link in the chain forwards `mime_type` and `metadata`.

## SKILL.md Optimisation

The SKILL.md is the single most impactful lever for agent performance. A well-written SKILL.md eliminates wasted tool calls, prevents dead-end approaches, and guides the agent to the fastest path.

### Principles

1. **Lead with capability, not prohibition.** Tell the agent what it CAN do natively before listing what to avoid. "Read PDFs directly with the Read tool" beats "Do NOT use pdftotext."

2. **Match instructions to mime_type.** Reference the resource index and route by file type:
   ```
   Check .eve/resources/index.json for file types.
   - application/pdf → Read natively (supports page ranges for large files)
   - text/* → Read directly
   - image/* → View with Read tool (multimodal)
   - audio/video → Describe, extract metadata, note for human review
   ```

3. **Specify page ranges for large documents.** Claude's Read tool supports `pages: "1-5"` for PDFs. For documents over 10 pages, process in chunks to avoid context overflow.

4. **Eliminate unnecessary tool dependencies.** If the LLM can process a file format natively (PDF, images, plain text), do not instruct it to shell out to conversion tools. Every unnecessary subprocess is latency and a potential failure point.

5. **Structure for scanning.** Agents scan SKILL.md before they read it deeply. Use clear headers, short paragraphs, and put the most important instruction for each section first.

6. **Include the output format.** Specify exactly what the agent should produce — JSON schema, markdown structure, or plain text. Ambiguity in output format wastes tokens on formatting decisions.

### Anti-Patterns

- **Listing every possible tool** — the agent already knows its tools. Only mention tools when the choice is non-obvious (e.g., "use Read, not pdftotext").
- **Verbose explanations** — agents don't need motivation. Short, direct instructions outperform essays.
- **Conditional logic trees** — keep branching minimal. If there are many file types, use a lookup table, not nested if/else prose.
- **Referencing tools by framework name** — say "read the file" not "use the Read tool." The harness maps to the right tool.

## Harness Configuration Tuning

### Model Selection

Choose the model based on task complexity, not habit:

| Task Type | Recommended Model | Why |
|-----------|-------------------|-----|
| Simple extraction, formatting | `haiku-4.5` | Fast, cheap, sufficient |
| General coding, analysis | `sonnet-4.6` (default) | Good balance of speed and capability |
| Complex reasoning, architecture | `opus-4-6` | Deepest reasoning, highest quality |
| Code generation with tools | `sonnet-4.6` or `codex` | Tool-use optimised |

Set via job creation:
```bash
eve job create --harness-options '{"model":"haiku-4.5"}' ...
```

Or via manifest defaults for an agent:
```yaml
x-eve:
  agents:
    doc-processor:
      harness_options:
        model: sonnet-4.6
        reasoning_effort: medium
```

### Reasoning Effort

Controls how many thinking tokens the model uses:

| Level | Thinking Tokens | When to Use |
|-------|----------------|-------------|
| `low` | ~1,024 | Simple tasks, known patterns, extraction |
| `medium` | ~8,192 | General work, moderate analysis (default) |
| `high` | ~16,000 | Complex problems, multi-step reasoning |
| `x-high` | ~32,000 | Expert-level analysis, novel problems |

Higher reasoning = slower + more expensive. Match to task complexity:
- Document extraction → `low`
- Code review → `medium`
- Architecture design → `high`
- Debugging obscure failures → `high` or `x-high`

### Permission Policy

Controls agent autonomy with file edits:

| Policy | Behaviour | Speed Impact |
|--------|-----------|--------------|
| `default` | Prompts for confirmation | Slowest (waits for input) |
| `auto_edit` | Auto-applies edits, prompts for other actions | Moderate |
| `yolo` | Auto-applies everything | Fastest |

For automated pipelines and batch processing, use `yolo`. For interactive or sensitive work, use `default` or `auto_edit`.

### Harness Variants

Custom configurations stored in the repo under `.agent/harnesses/{harness}/variants/{name}/`:

```
.agent/harnesses/mclaude/variants/fast/
  config.json    # {"model": "haiku-4.5", "reasoning_effort": "low"}
```

Reference in job creation:
```bash
eve job create --harness-options '{"variant":"fast"}' ...
```

Variants let different job types use different configs without changing the default.

## Performance Patterns

### 1. Workspace Reuse for Iterative Work

```bash
# Session mode: workspace persists across attempts
eve job create --workspace-mode session --workspace-key "sprint-42" ...
```

Avoids re-cloning the repo on every attempt. Use for iterative work where the agent builds on previous state.

### 2. Right-Size the Toolchain

Only request toolchains the agent actually needs:

```yaml
x-eve:
  agents:
    my-agent:
      toolchains: [python]  # Only Python, not the full stack
```

Each toolchain adds init container startup time. An agent that only reads files doesn't need language runtimes.

### 3. Minimise Resource Refs

Only attach resources the agent will actually use. Every resource ref triggers a download during hydration. Large files (videos, archives) add significant provisioning latency.

### 4. Use Attachments for Structured Output

Instead of parsing agent prose for results, instruct the agent to write structured output:

```bash
eve job attach $EVE_JOB_ID --name result.json --content '{"findings": [...]}'
```

Parent jobs can then read child attachments directly — no parsing, no ambiguity.

### 5. Budget Enforcement

Set cost limits to catch runaway agents:

```bash
eve job create --max-cost '{"currency":"USD","amount":"0.50"}' ...
```

The harness is killed if budget is exceeded. Better to fail fast than accumulate unexpected costs.

## Common Failure Modes

### Agent Can't Find Its Resources

**Symptom**: Agent logs show "file not found" for expected inputs.

**Diagnosis**:
```bash
eve job show <id> --verbose  # Check resource_refs status
```

**Causes**:
- Resource hydration failed silently (storage credentials).
- `mime_type` not threaded through pipeline — agent doesn't know what to look for.
- `mount_path` collision — two resources mapped to the same path.

**Fix**: Verify the resource index exists in the workspace and all entries show `status: resolved`.

### Agent Uses Wrong Tool for File Type

**Symptom**: Agent shells out to `pdftotext` or `ffmpeg` when the LLM can process natively.

**Diagnosis**: Check SKILL.md for explicit file-type routing instructions.

**Fix**: Update SKILL.md to lead with native capabilities. For PDFs: "Read directly with the Read tool. Do NOT use conversion tools." For images: "View directly — the LLM is multimodal."

### Agent Runs Out of Context

**Symptom**: Agent output becomes repetitive, loses track of earlier work, or produces truncated results.

**Diagnosis**: Check token usage in job result. Compare against model context window.

**Fix**:
- Reduce reasoning effort (fewer thinking tokens consumed).
- Process large documents in page ranges instead of reading entire file.
- Split the job into smaller children via orchestration.
- Use a model with a larger context window.

### Slow Provisioning

**Symptom**: Long delay between job claimed and job active.

**Diagnosis**:
```bash
eve job diagnose <id>  # Check phase-latency waterfall
```

**Causes**:
- Large repo clone (use sparse checkout or shallow clone via git controls).
- Many/large resource downloads (reduce resource_refs to essentials).
- Toolchain init containers (remove unused toolchains).
- Cold pod startup (agent-runtime warm pods mitigate this).

### Storage Credential Mismatch

**Symptom**: Resource hydration fails with `InvalidAccessKeyId` or `SignatureDoesNotMatch`.

**Diagnosis**: Compare storage credentials in K8s secrets vs actual storage backend (MinIO/S3).

**Fix**: Ensure `EVE_STORAGE_ACCESS_KEY` and `EVE_STORAGE_SECRET_KEY` in system secrets match the storage backend's root credentials. After fixing, roll out all pods to pick up new secrets.

### AWS SDK Compatibility with MinIO

**Symptom**: `NotImplemented` errors on bucket operations (CORS, lifecycle).

**Cause**: AWS SDK v3 sends flexible checksum headers that MinIO doesn't support.

**Fix**: Ensure the S3 client is configured with:
```typescript
requestChecksumCalculation: 'WHEN_REQUIRED',
responseChecksumValidation: 'WHEN_REQUIRED',
```

Make non-essential operations (CORS setting) non-fatal so they don't block the primary flow.

## Optimisation Checklist

Use this checklist when an agent isn't performing well:

- [ ] **Check job result** — exit code, token usage, cost, duration.
- [ ] **Stream logs** — identify where time is spent, what tool calls happen.
- [ ] **Verify resources** — all required resources resolved in index.json.
- [ ] **Review SKILL.md** — clear instructions, native capabilities first, output format specified.
- [ ] **Check model/reasoning** — right-sized for the task complexity.
- [ ] **Check permission policy** — `yolo` for automated work, `default` for interactive.
- [ ] **Check toolchains** — only what's needed, no extras.
- [ ] **Check workspace mode** — `session` for iterative, `job` for one-shot.
- [ ] **Check budget** — set and reasonable for the task.
- [ ] **Check harness routing** — agent jobs go to agent-runtime, not worker.

## Related Skills

- `eve-job-debugging` — CLI commands for monitoring and diagnosing jobs.
- `eve-orchestration` — decomposing work into parallel children.
- `eve-agent-memory` — storage primitives for agent persistence.
- `eve-skill-distillation` — encoding learned patterns into reusable skills.
- `eve-platform-debugging` — deep platform-level diagnostics when CLI isn't enough.
