---
name: netlify-functions
description: Correct implementation of Netlify Functions with modern TypeScript syntax. Use when creating API endpoints for Vite + React apps or mutation operations in Astro apps. Covers the modern default export pattern, Config type, path configuration, and common mistakes agents make with the syntax.
---

# Netlify Functions

Modern Netlify Functions implementation with TypeScript.

---

## Critical: Use Modern Syntax

Agents frequently get this wrong. Here's the correct pattern:

```typescript
// netlify/functions/example.ts
import type { Context, Config } from "@netlify/functions";

export default async (request: Request, context: Context) => {
  return new Response("Hello World");
};

export const config: Config = {
  path: "/api/example",
};
```

### What NOT to Do

```typescript
// ❌ WRONG: Legacy CommonJS syntax
exports.handler = async (event, context) => {
  return { statusCode: 200, body: "Hello" };
};

// ❌ WRONG: Named handler export
export const handler = async (event, context) => { ... };

// ❌ WRONG: Express-style response
return res.status(200).json({ data });
```

---

## Function Structure

### Default Export (Handler)

The handler receives standard Web API `Request` and Netlify `Context`:

```typescript
export default async (request: Request, context: Context) => {
  // request: Standard Web API Request object
  // context: Netlify-specific context (params, geo, etc.)

  return new Response("Response body");
};
```

### Config Export (Path Configuration)

Define the route **inside the function file**:

```typescript
export const config: Config = {
  path: "/api/users",
  method: "GET",
};
```

This way, you open the file and immediately see its route.

---

## Config Options

```typescript
export const config: Config = {
  // Static path
  path: "/api/items",

  // Path with parameter
  path: "/api/items/:id",

  // Multiple parameters
  path: "/api/posts/:postId/comments/:commentId",

  // Catch-all
  path: "/api/*",

  // Single method
  method: "GET",

  // Multiple methods
  method: ["GET", "POST", "PUT", "DELETE"],
};
```

---

## Request Handling

### Reading URL and Params

```typescript
export default async (request: Request, context: Context) => {
  // URL info
  const url = new URL(request.url);

  // Query parameters
  const page = url.searchParams.get("page") || "1";
  const limit = url.searchParams.get("limit") || "10";

  // Path parameters (from config path)
  const { id } = context.params;

  return Response.json({ id, page, limit });
};

export const config: Config = {
  path: "/api/items/:id",
  method: "GET",
};
```

### Reading Request Body

```typescript
export default async (request: Request, context: Context) => {
  // JSON body
  const body = await request.json();

  // Form data
  const formData = await request.formData();
  const name = formData.get("name");

  // Raw text
  const text = await request.text();

  return Response.json({ received: body });
};
```

### Reading Headers

```typescript
export default async (request: Request, context: Context) => {
  const authHeader = request.headers.get("Authorization");
  const contentType = request.headers.get("Content-Type");

  // Custom headers (e.g., from auth proxy)
  const userId = request.headers.get("x-user-id");

  return new Response("OK");
};
```

---

## Response Patterns

### JSON Response

```typescript
return Response.json({ id: 1, name: "Item" });

// With status code
return Response.json({ error: "Not found" }, { status: 404 });

// With headers
return Response.json(data, {
  status: 200,
  headers: {
    "Cache-Control": "public, max-age=3600",
  },
});
```

### Plain Text

```typescript
return new Response("Hello World");

return new Response("Not found", { status: 404 });
```

### Redirect (Astro API routes typically return this)

```typescript
return Response.redirect(new URL("/success", request.url), 302);

// Or using the request URL
const url = new URL(request.url);
return Response.redirect(`${url.origin}/success?message=done`, 302);
```

### Stream Response

```typescript
return new Response(readableStream, {
  headers: {
    "Content-Type": "application/octet-stream",
  },
});
```

---

## Method Routing

Handle multiple methods in one function:

```typescript
export default async (request: Request, context: Context) => {
  const { id } = context.params;

  switch (request.method) {
    case "GET":
      return handleGet(id);
    case "PUT":
      return handlePut(id, await request.json());
    case "DELETE":
      return handleDelete(id);
    default:
      return new Response("Method not allowed", { status: 405 });
  }
};

export const config: Config = {
  path: "/api/items/:id",
  method: ["GET", "PUT", "DELETE"],
};
```

---

## Full CRUD Example

```typescript
// netlify/functions/items.ts
import type { Context, Config } from "@netlify/functions";
import { db, items } from "./_shared/db";
import { eq } from "drizzle-orm";
import { requireAuth } from "./_shared/auth";

export default async (request: Request, context: Context) => {
  const { id } = context.params;

  // GET /api/items or GET /api/items/:id
  if (request.method === "GET") {
    if (id) {
      const [item] = await db
        .select()
        .from(items)
        .where(eq(items.id, parseInt(id)))
        .limit(1);

      if (!item) {
        return Response.json({ error: "Not found" }, { status: 404 });
      }
      return Response.json(item);
    }

    const allItems = await db.select().from(items);
    return Response.json(allItems);
  }

  // Auth required for mutations
  const auth = await requireAuth(request);
  if (!auth.authenticated) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  // POST /api/items
  if (request.method === "POST") {
    const body = await request.json();
    const [newItem] = await db.insert(items).values(body).returning();
    return Response.json(newItem, { status: 201 });
  }

  // PUT /api/items/:id
  if (request.method === "PUT") {
    if (!id) {
      return Response.json({ error: "ID required" }, { status: 400 });
    }

    const body = await request.json();
    const [updated] = await db
      .update(items)
      .set({ ...body, updatedAt: new Date() })
      .where(eq(items.id, parseInt(id)))
      .returning();

    if (!updated) {
      return Response.json({ error: "Not found" }, { status: 404 });
    }
    return Response.json(updated);
  }

  // DELETE /api/items/:id
  if (request.method === "DELETE") {
    if (!id) {
      return Response.json({ error: "ID required" }, { status: 400 });
    }

    await db.delete(items).where(eq(items.id, parseInt(id)));
    return Response.json({ success: true });
  }

  return new Response("Method not allowed", { status: 405 });
};

export const config: Config = {
  path: ["/api/items", "/api/items/:id"],
  method: ["GET", "POST", "PUT", "DELETE"],
};
```

---

## Shared Utilities

Organize shared code in a `_shared` directory:

```
netlify/functions/
├── _shared/
│   ├── auth.ts
│   ├── db.ts
│   └── utils.ts
├── items.ts
├── users.ts
└── upload.ts
```

```typescript
// netlify/functions/_shared/db.ts
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';
import * as schema from '../../../db/schema';

const sql = neon(process.env.NETLIFY_DATABASE_URL!);
export const db = drizzle(sql, { schema });
export { schema };
```

```typescript
// netlify/functions/_shared/auth.ts
export async function requireAuth(request: Request) {
  const userId = request.headers.get("x-user-id");
  const email = request.headers.get("x-user-email");

  if (!userId || !email) {
    return { authenticated: false };
  }

  return { authenticated: true, userId, email };
}
```

---

## File Location

Functions go in `netlify/functions/`:

```
project/
├── netlify/
│   └── functions/
│       ├── _shared/
│       │   └── ...
│       └── example.ts
├── src/
└── netlify.toml
```

Configure in `netlify.toml`:

```toml
[build]
  functions = "netlify/functions"
```

---

## Anti-Patterns

- **Using legacy `exports.handler` syntax** - Use default export
- **Using Express-style `res.json()`** - Use `Response.json()`
- **Not configuring path in function file** - Always export config
- **Putting path in netlify.toml** - Put it in the function file
- **Using `.mjs` extension** - Use `.ts` with TypeScript

---

## Related Skills

- `vite-best-practices` - Integrating functions with React
- `astro-best-practices` - When to use functions vs API routes
- `auth-design` - Protecting function endpoints
- `data-storage` - Database access in functions
