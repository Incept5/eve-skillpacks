---
name: eve-skill-distillation
description: Distill repeated work into Eve skillpacks by creating or updating skills with concise instructions and references. Use when a workflow repeats or knowledge should be shared across agents.
---

# Eve Skill Distillation

Use this workflow to turn repeated patterns into reusable skills.

## Capture the pattern

- Identify repeated steps, commands, or failure modes.
- Decide whether to update an existing skill or create a new one.
- Choose the pack:
  - `private-eve-dev-skills/eve-dev` for platform development (private)
  - `eve-se` for Eve-compatible project work
  - `eve-work` for general knowledge work

## Create or update the skill

- Add a new skill directory with a `SKILL.md` file.
- Use YAML frontmatter with `name` and `description` only.
- Write instructions in imperative form; keep the body concise.
- Move long details into a `references/` file if needed.

## Validate and publish

- Use the skill in the next relevant job.
- Update pack README and `ARCHITECTURE.md` listings (or the private pack README for dev-only skills).
- Sync or reinstall skills if needed (`openskills install` and `openskills sync`).

## Recursive skill distillation

- Repeat this loop after each significant job.
- Merge overlapping skills instead of duplicating them.
- Keep skills current as platform behavior evolves.
