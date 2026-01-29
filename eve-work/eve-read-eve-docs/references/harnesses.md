# Harnesses + Policies (Current)

## Supported Harnesses

- **mclaude / claude / zai** (Anthropic-based)
- **codex / code** (OpenAI)
- **gemini** (Google)

## Job-Level Harness Fields

```json
{
  "harness": "mclaude",
  "harness_profile": "primary-reviewer",
  "harness_options": {
    "variant": "deep",
    "model": "opus-4.5",
    "reasoning_effort": "high"
  },
  "hints": {
    "worker_type": "default",
    "permission_policy": "auto_edit",
    "timeout_seconds": 1800
  }
}
```

Notes:
- Use `harness_options.variant` (not `harness:variant`).
- `hints` are for scheduling (worker type, permission, timeout) only.

## Project Harness Profiles (Manifest)

```yaml
x-eve:
  agents:
    version: 1
    availability:
      drop_unavailable: true
    profiles:
      primary-reviewer:
        - harness: mclaude
          model: opus-4.5
          reasoning_effort: high
        - harness: codex
          model: gpt-5.2-codex
          reasoning_effort: x-high
```

Use profiles in jobs (`--profile primary-reviewer`) to avoid hardcoding harness choices.

## Auth Priority (Current)

- **Claude/mclaude/zai**: `ANTHROPIC_API_KEY` (highest) -> OAuth tokens
- **Codex/Code**: `OPENAI_API_KEY` or `CODEX_AUTH_JSON_B64`
- **Gemini**: `GEMINI_API_KEY` or `GOOGLE_API_KEY`
- **Z.ai**: `Z_AI_API_KEY`

## Sandbox Flags (Security)

- Claude/mclaude/zai: `--add-dir <workspace>`
- Codex/Code: `--sandbox workspace-write -C <workspace>`
- Gemini: `--sandbox`

These are applied automatically by `eve-agent-cli` for shared-worker safety.
