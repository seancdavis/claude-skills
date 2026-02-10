---
name: astro-best-practices
description: Conventions and patterns for Astro projects deployed to Netlify. Use when building Astro sites, configuring the Netlify adapter, organizing pages and components, implementing React islands, or handling server-side data fetching. Covers client directives, component strategy, API routes, and SSR patterns.
---

# Astro Best Practices

Conventions and patterns for building Astro applications deployed to Netlify.

---

## Core Principles

1. **Server-first rendering** - Render as much as possible on the server
2. **Minimal JavaScript** - Only ship JS when interactivity requires it
3. **React for components** - Use React components (not Astro components) for reusability
4. **Progressive enhancement** - Pages should work without JavaScript when possible

---

## Project Configuration

### astro.config.mjs

```javascript
import { defineConfig } from "astro/config";
import netlify from "@astrojs/netlify";
import react from "@astrojs/react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  output: "server",
  adapter: netlify(),
  integrations: [react()],
  vite: {
    plugins: [tailwindcss()],
  },
});
```

### Local Development

Always use `netlify dev` to run the Astro dev server - this brings in the Netlify environment (environment variables, blobs, database):

```json
{
  "scripts": {
    "dev": "netlify dev --no-open"
  }
}
```

The `--no-open` flag prevents auto-opening the browser.

### netlify.toml

```toml
[build]
  command = "npm run build"
  publish = "dist"

[dev]
  command = "astro dev"
  targetPort = 4321
```

---

## Component Strategy

### Use React Components, Not Astro Components

**Why:** React components can later become interactive islands without rewriting. Astro components (.astro) require a rewrite if you need to add interactivity.

```tsx
// src/components/ui/Button.tsx - Can become interactive later
export function Button({ children, onClick }: ButtonProps) {
  return <button onClick={onClick}>{children}</button>;
}
```

```astro
<!-- src/pages/index.astro - Astro orchestrates -->
---
import { Button } from "../components/ui/Button";
---
<Button>Click me</Button>
```

### Astro Pages as Orchestrators

Astro pages and layouts:
- Handle routing (file-based)
- Fetch data server-side
- Compose React components
- Control where `client:*` directives are applied

```astro
---
// src/pages/dashboard.astro
import Layout from "../layouts/Layout.astro";
import { DashboardHeader } from "../components/DashboardHeader";
import { DashboardContent } from "../components/DashboardContent";
import { getUserData } from "../lib/data";

const data = await getUserData(Astro.request);
---
<Layout title="Dashboard">
  <DashboardHeader user={data.user} />
  <DashboardContent items={data.items} client:load />
</Layout>
```

---

## Client Directives

Push client directives **as far down the component tree as possible** to minimize JavaScript.

### Directive Reference

| Directive | When to Use |
|-----------|-------------|
| `client:load` | Interactive immediately on page load |
| `client:idle` | Interactive after page becomes idle |
| `client:visible` | Interactive when scrolled into view |
| `client:media` | Interactive at specific breakpoint |
| `client:only="react"` | Never SSR, client-only |

### Bad: Directive on Layout Component

```astro
<!-- DON'T: Entire header is now client-side -->
<Header client:load />
```

### Good: Directive on Interactive Child

```astro
<!-- DO: Only the interactive part ships JS -->
<header>
  <Logo />
  <nav>...</nav>
  <UserMenu client:load />
</header>
```

### Example: Breaking Down a Header

```astro
---
// src/layouts/Layout.astro
import { Logo } from "../components/ui/Logo";
import { NavMenu } from "../components/auth/NavMenu";
import { UserMenu } from "../components/auth/UserMenu";

const { user } = Astro.props;
---
<header>
  <Logo />
  <NavMenu />
  {user ? <UserMenu user={user} client:load /> : <a href="/login">Sign in</a>}
</header>
```

The `Logo` and `NavMenu` are static - no JS. Only `UserMenu` (with dropdown, logout button) needs interactivity.

---

## Page Patterns

### Authenticated Page

```astro
---
// src/pages/profile.astro
import Layout from "../layouts/Layout.astro";
import { ProfilePage } from "../components/pages/ProfilePage";
import { getUserWithApproval } from "../lib/auth";

const auth = await getUserWithApproval(Astro.request);

if (!auth) {
  return Astro.redirect("/login");
}

if (!auth.isApproved) {
  return Astro.redirect("/unauthorized");
}

const { user, isAdmin } = auth;
---
<Layout title="Profile">
  <ProfilePage user={user} isAdmin={isAdmin} />
</Layout>
```

### Data Fetching in Frontmatter

```astro
---
// src/pages/posts/[id].astro
import Layout from "../../layouts/Layout.astro";
import { PostPage } from "../../components/pages/PostPage";
import { getPostById } from "../../lib/posts";

const { id } = Astro.params;
const post = await getPostById(id);

if (!post) {
  return Astro.redirect("/404");
}
---
<Layout title={post.title}>
  <PostPage post={post} />
</Layout>
```

---

## API Routes

API routes handle mutations and return **redirects with feedback message keys**.

### Pattern: Form Submission

```typescript
// src/pages/api/entries.ts
import type { APIRoute } from "astro";
import { getUserWithApproval } from "../../lib/auth";
import { createEntry } from "../../lib/entries";
import { logger } from "../../lib/logger";

const log = logger.scope("ENTRIES");

export const POST: APIRoute = async ({ request, redirect }) => {
  // Auth check
  const auth = await getUserWithApproval(request);
  if (!auth) {
    return redirect("/login?message=auth_error", 302);
  }
  if (!auth.isApproved) {
    return redirect("/unauthorized?message=unauthorized", 302);
  }

  // Parse form data
  const formData = await request.formData();
  const title = formData.get("title")?.toString().trim();
  const description = formData.get("description")?.toString().trim();

  // Validate
  if (!title || !description) {
    log.warn("Missing required fields");
    return redirect("/entries/new?message=validation_error", 302);
  }

  // Create
  try {
    await createEntry({ title, description, userEmail: auth.user.email });
    log.info("Entry created for:", auth.user.email);
    return redirect("/entries?message=entry_created", 302);
  } catch (error) {
    log.error("Failed to create entry:", error);
    return redirect("/entries/new?message=create_failed", 302);
  }
};
```

### Method Override Pattern

For PUT/DELETE from HTML forms (which only support GET/POST):

```typescript
export const POST: APIRoute = async ({ request, redirect }) => {
  const formData = await request.formData();
  const method = formData.get("_method")?.toString() || "POST";

  if (method === "DELETE") {
    return handleDelete(formData, redirect);
  }
  if (method === "PUT") {
    return handleUpdate(formData, redirect);
  }
  return handleCreate(formData, redirect);
};
```

```html
<form action="/api/entries" method="post">
  <input type="hidden" name="_method" value="DELETE" />
  <input type="hidden" name="id" value={entry.id} />
  <button type="submit">Delete</button>
</form>
```

---

## Folder Structure

```
src/
├── components/
│   ├── auth/          # Auth-related (LoginButton, UserMenu)
│   ├── layout/        # Header, Footer, Nav
│   ├── pages/         # Page-level React components
│   ├── ui/            # Reusable UI (Button, Card, Toast)
│   └── {feature}/     # Feature-specific components
├── layouts/
│   └── Layout.astro   # Main layout
├── lib/
│   ├── auth.ts        # Auth utilities
│   ├── logger.ts      # Logging utility
│   ├── messages.ts    # Feedback message constants
│   └── {feature}.ts   # Feature-specific utilities
├── pages/
│   ├── api/           # API routes
│   ├── index.astro
│   └── {routes}.astro
└── db/
    └── index.ts       # Database connection and schema exports
```

---

## Server Islands (Advanced)

Use Server Islands for forms on pre-rendered pages that need server interaction without navigating away:

```astro
---
// A server island in a static page
---
<ContactForm server:defer />
```

The form renders on the server, handles submissions server-side, and can update without a full page navigation.

---

## Anti-Patterns

- **Don't use Astro components for anything that might need interactivity** - Rewriting .astro to .tsx is costly
- **Don't put client directives on layout components** - This hydrates the entire subtree
- **Don't render JSON from API routes** - Return redirects; clients should land on pages
- **Don't fetch data in React components** - Fetch in Astro frontmatter, pass as props
- **Don't use `npm run dev` directly** - Always use `netlify dev --no-open`

---

## Related Skills

- `auth-design` - Authentication patterns for Astro
- `routing-design` - URL structure conventions
- `feedback` - Toast and message patterns
- `forms` - Form submission patterns
- `component-design` - Component organization
