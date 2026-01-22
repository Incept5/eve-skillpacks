---
name: eve-job-debugging
description: Monitor and debug Eve jobs with CLI follow, logs, wait, and diagnose commands. Use when work is stuck, failing, or you need fast status.
---

# Eve Job Debugging

Use CLI-first diagnostics before digging into service logs.

## Monitor

- `eve job follow <id>` to stream logs.
- `eve job wait <id> --timeout 300 --json` to wait on completion.
- `eve job result <id> --format text` for the latest result.

## Diagnose

- `eve job diagnose <id>` for timeline and error summary.
- `eve job show <id> --verbose` for attempts and phase.
- `eve job dep list <id>` for dependency blocks.

## System health

- `eve system health` to confirm the API is reachable.

## Recursive skill distillation

- Add new debugging patterns and failure signatures as they appear.
- Split advanced diagnosis into a new skill if this grows too large.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
