---
name: eve-docs-upkeep
description: Keep Eve Horizon documentation current, prune stale docs, and maintain cross-links. Use when editing docs or when behavior changes require documentation updates.
---

# Eve Docs Upkeep

Keep docs aligned with current platform behavior.

## Workflow

1. Scan recent changes and identify impacted docs.
2. Update top-level docs: `README.md`, `AGENTS.md`, `docs/system/README.md`.
3. Review `docs/ideas` for outdated material; delete or flag if unclear.
4. Update cross-links and verify referenced files exist.
5. Append the `AGENTS.md` update log for significant doc changes.

## Rules

- Prefer updating system docs over duplicating content.
- Remove outdated docs rather than preserving speculation.
- Call out ambiguities before deleting borderline docs.

## Recursive skill distillation

- Record new doc maintenance patterns here as they emerge.
- Create a separate eve-dev skill if upkeep grows beyond this scope.
- Update the eve-skillpacks README and ARCHITECTURE listings after changes.
