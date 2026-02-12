---
name: onboard-existing-project
description: Onboards an existing project to use the seancdavis-skills plugin. Use after installing the plugin in an existing codebase. Adds skills reference to CLAUDE.md, audits for conflicting patterns, and updates or removes outdated instructions. Invoke with `/onboard-existing-project` after running `/plugin install seancdavis-skills`.
---

# Onboard Existing Project

This skill integrates the seancdavis-skills plugin into an existing project's CLAUDE.md and resolves conflicts.

---

## Step 1: Identify Project Technologies

Read the project's configuration files to determine which skills are relevant:

| Check for...                        | Indicates skill          |
| ----------------------------------- | ------------------------ |
| `astro.config.*`                    | `astro-best-practices`   |
| `vite.config.*` (without Astro)     | `vite-best-practices`    |
| `@astrojs/netlify` or `netlify.toml`| `netlify-functions`      |
| `drizzle.config.*` or `db/schema.*` | `data-storage`           |
| Neon Auth imports or `neon_auth`    | `auth-design`            |
| `@netlify/blobs` imports            | `file-storage`           |
| Toast/notification components       | `feedback`               |
| Form handling code                  | `forms`                  |
| Tailwind config                     | `ui-design`              |

---

## Step 2: Add Skills Section to CLAUDE.md

If no CLAUDE.md exists, create one with minimal content. If one exists, add the skills section.

### Minimal CLAUDE.md (if none exists)

```markdown
# {PROJECT_NAME}

## Skills

This project uses the `seancdavis-skills` plugin. When skill instructions conflict with patterns described elsewhere in this file, defer to the skills.

Relevant skills:
{SKILLS_LIST}
```

### Skills section to add (if CLAUDE.md exists)

Add this section near the top, after any project description but before detailed conventions:

```markdown
## Skills

This project uses the `seancdavis-skills` plugin. When skill instructions conflict with patterns described elsewhere in this file, defer to the skills.

Relevant skills:
- `{skill-name}` - {brief description}
```

---

## Step 3: Audit for Conflicts

Review the existing CLAUDE.md for patterns that conflict with skills. Common conflicts:

### Routing patterns
- **Conflict:** Custom URL conventions that differ from `routing-design` skill
- **Resolution:** Remove custom routing section, or keep project-specific exceptions only

### Database patterns
- **Conflict:** Different ORM setup, migration commands, or schema organization
- **Resolution:** Update to match `data-storage` skill patterns, or note exceptions

### Authentication
- **Conflict:** Different auth provider or flow than Neon Auth
- **Resolution:** If using Neon Auth, remove old instructions. If different auth, note as exception.

### Form handling
- **Conflict:** Different form submission patterns (e.g., always JSON, always HTTP)
- **Resolution:** Update to match `forms` skill (Astro=HTTP, React=JSON)

### Component organization
- **Conflict:** Different folder structure or naming conventions
- **Resolution:** Keep project-specific structure if established, note as exception

### Commit conventions
- **Conflict:** Different commit message format
- **Resolution:** Keep existing if team has established conventions

---

## Step 4: Update or Remove Conflicting Sections

For each conflict found:

1. **If skill should take precedence:** Remove the conflicting section entirely
2. **If project has legitimate exceptions:** Keep the section but add a note:
   ```markdown
   > **Note:** This differs from the `{skill-name}` skill. This project uses {reason}.
   ```
3. **If section is outdated:** Remove it

### Sections safe to remove (skill covers them)

- Generic Astro/Vite setup instructions
- Boilerplate Netlify deployment info
- Standard Drizzle/database commands
- Common toast/feedback patterns
- Basic form handling instructions

### Sections to keep (project-specific)

- Project-specific business logic
- Custom API endpoints unique to this project
- Team conventions that differ from skills
- Third-party integrations not covered by skills

---

## Step 5: Verify Integration

After updates:

1. **Read the updated CLAUDE.md** to ensure it's coherent
2. **Check that skills section is present** with relevant skills listed
3. **Confirm no duplicate instructions** between CLAUDE.md and skills

---

## Example Output

For an Astro project with Netlify DB and Neon Auth:

```markdown
## Skills

This project uses the `seancdavis-skills` plugin. When skill instructions conflict with patterns described elsewhere in this file, defer to the skills.

Relevant skills:
- `astro-best-practices` - Astro patterns, React islands, SSR
- `auth-design` - Neon Auth with Google OAuth
- `data-storage` - Netlify DB + Drizzle ORM
- `routing-design` - Rails-style CRUD routes
- `feedback` - Toast notifications via query params
- `forms` - HTTP form submissions with redirects
- `netlify-functions` - API endpoints for mutations
- `logging-and-monitoring` - Three-level server logging
```

---

## Checklist

- [ ] Project technologies identified
- [ ] Relevant skills determined
- [ ] Skills section added to CLAUDE.md
- [ ] Existing CLAUDE.md audited for conflicts
- [ ] Conflicting sections removed or annotated
- [ ] Final CLAUDE.md is coherent and non-redundant
