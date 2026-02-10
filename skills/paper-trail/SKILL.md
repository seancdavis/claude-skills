---
name: paper-trail
description: Maintaining a decision log alongside Git history for capturing the "why" behind decisions. Use when documenting architectural decisions, recording trade-offs, or establishing project conventions. Follows ADR (Architecture Decision Records) patterns.
---

# Paper Trail

Maintaining a decision log alongside Git history.

---

## Purpose

Git commits capture **what** happened. This skill captures **why** decisions were made.

---

## When to Create a Record

- Major architectural decisions (framework choice, database design)
- Technology selections with trade-offs
- Patterns established that aren't obvious
- Decisions that might be questioned later
- Changes from original plan

---

## Location

Store in `docs/decisions/` (not `.claude/` — should be public and tracked):

```
project/
├── docs/
│   └── decisions/
│       ├── 0001-use-astro-over-nextjs.md
│       ├── 0002-timestamp-migrations.md
│       └── 0003-approved-users-safelist.md
├── src/
└── ...
```

---

## ADR Format

Use Architecture Decision Records (ADR) format:

```markdown
# {NUMBER}. {TITLE}

Date: {YYYY-MM-DD}
Status: {proposed | accepted | deprecated | superseded by [ADR-XXXX]}

## Context

{What is the issue or question that motivated this decision?}
{What constraints or forces are at play?}

## Decision

{What is the change or solution being proposed/made?}
{Be specific and concrete.}

## Consequences

{What becomes easier or harder as a result?}
{What are the trade-offs?}
{Are there follow-up actions needed?}
```

---

## Example: Framework Selection

```markdown
# 0001. Use Astro over Next.js

Date: 2024-01-15
Status: accepted

## Context

Starting a new Super Bowl party app. Users will view entries, vote, and check
a squares game. The vast majority of interactions are read-only displays. Only
a few forms for creating entries and casting votes.

Options considered:
- Next.js (App Router)
- Astro with React islands
- Vite + React SPA

## Decision

Use Astro with React components and server-side rendering.

Reasons:
- Most pages are content display (server rendering is ideal)
- Only 2-3 interactive components (form submission, squares picker)
- Ships minimal JavaScript by default
- Better performance for mobile users with spotty Super Bowl party WiFi
- Simpler mental model: server by default, client when needed

## Consequences

**Easier:**
- SEO and performance are handled automatically
- No client-side routing complexity for mostly-static pages
- Faster initial page loads

**Harder:**
- Need to think about which components need `client:*` directives
- Can't use React context across the whole app (islands are isolated)
- Some patterns from SPA world don't apply

**Follow-up:**
- Use React components (not Astro components) for anything that might
  eventually need interactivity
```

---

## Example: Technical Decision

```markdown
# 0002. Use Timestamp Prefixes for Migrations

Date: 2024-01-16
Status: accepted

## Context

Drizzle ORM defaults to sequential migration numbering (0001, 0002, etc.).
We have multiple developers and may work on parallel branches that each
create migrations.

Sequential numbering causes conflicts when merging branches that both
added a "0003" migration.

## Decision

Configure Drizzle to use Unix timestamp prefixes for migrations:

```typescript
// drizzle.config.ts
export default defineConfig({
  migrations: {
    prefix: 'timestamp'
  }
});
```

This produces migrations like `20240116123456_add_users_table.sql`.

## Consequences

**Easier:**
- Parallel branches can both create migrations without conflicts
- Clear ordering by creation time
- Safer for team development

**Harder:**
- Migration filenames are longer
- Can't easily see "how many migrations" at a glance

The trade-off is worth it for the merge safety.
```

---

## Lightweight Format

For smaller decisions, a lighter format works:

```markdown
# 0003. Approved Users Safelist Pattern

Date: 2024-01-16
Status: accepted

**Context:** Need auth but can't prevent Google OAuth signups.

**Decision:** Use `approved_users` database table as safelist. Users who
sign in but aren't in the table see an "unauthorized" page.

**Rationale:** Simpler than trying to restrict OAuth at the provider level.
Allows adding users by just inserting rows.
```

---

## When NOT to Write an ADR

- Obvious choices (TypeScript over JavaScript)
- Temporary workarounds (note in code instead)
- Implementation details that don't affect architecture
- Decisions already well-documented elsewhere

---

## Updating Records

Don't delete old records. Instead:

```markdown
Status: superseded by [0007-switch-to-supabase.md]
```

Or:

```markdown
Status: deprecated

## Update (2024-03-01)

This approach was abandoned because [reason]. See 0007 for current approach.
```

---

## Naming Convention

```
{NNNN}-{short-description}.md

0001-use-astro-framework.md
0002-timestamp-migrations.md
0003-approved-users-safelist.md
```

Keep numbers sequential within the project (unlike migrations).

---

## Template

```markdown
# {NUMBER}. {TITLE}

Date: {YYYY-MM-DD}
Status: accepted

## Context

{1-3 paragraphs explaining the situation}

## Decision

{1-2 paragraphs explaining what we decided}

## Consequences

**Easier:**
- {benefit}
- {benefit}

**Harder:**
- {trade-off}
- {trade-off}
```

---

## Anti-Patterns

- **Writing ADRs after the fact** - Write them when making the decision
- **Too much detail** - Keep it scannable; link to external docs if needed
- **No consequences section** - Trade-offs are the most valuable part
- **Deleting old records** - Supersede them instead
- **Storing in .claude/** - Should be in version control, visible to all

---

## Related Skills

- `new-project` - Record framework selection decisions
- `data-storage` - Record database schema decisions
- `auth-design` - Record authentication approach decisions
