# Claude Skills Repository

A collection of reusable Claude Code agent skills for common development workflows.

## Project Structure

```
skills/
  <skill-name>/
    SKILL.md      # Required: Frontmatter + instructions
    scripts/      # Optional: Executable code
    references/   # Optional: Documentation for context
    assets/       # Optional: Templates, icons, etc.
```

## Agent Skills Standard

All skills must follow the [Agent Skills spec](https://github.com/anthropics/skills):

### Required SKILL.md Frontmatter

```yaml
---
name: skill-name
description: What the skill does and when to use it. Be comprehensive—this is how Claude decides to invoke the skill.
---
```

### Optional Frontmatter Fields

- `disable-model-invocation: true` - Only user can invoke (for side-effect workflows)
- `user-invocable: false` - Only Claude can invoke (for background knowledge)
- `allowed-tools` - Restrict tools available when skill is active

### Guidelines

- Keep SKILL.md under 500 lines
- Move detailed reference material to `references/` directory
- Description must include all trigger conditions (body loads only after triggering)

## Development Workflow

- Commit after each meaningful change
- Use Prettier for formatting
