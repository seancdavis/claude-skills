---
name: auth-design
description: Authentication patterns and implementation for Netlify projects using Neon Auth with Google OAuth. Use when implementing user authentication, protecting routes/pages, managing sessions, or implementing user safelists. Covers three tiers of access control, server-side auth patterns, and common pitfalls agents encounter with auth.
---

# Auth Design

## Three Tiers of Access Control

### Tier 1: Personal/Private Apps (Simplest)

For apps only Sean uses:

**Solution:** Mark the Netlify site as private → uses Netlify team SSO.

No auth code needed. Netlify handles everything.

### Tier 2: Apps with Other Users

For apps where specific people need access:

**Solution:** Neon Auth + approved users safelist.

Since Google OAuth signups can't be prevented, implement an `approved_users` table. Unapproved users see an unauthorized page.

### Tier 3: Machine-to-Machine / Public APIs

For services calling your API:

**Solution:** Simple API key approach.

Store API key in Netlify environment variables, validate in function handlers.

---

## Neon Auth Setup

### Environment Configuration

Neon Auth requires a proxy redirect in `netlify.toml`:

```toml
[[redirects]]
  from = "/neon-auth/*"
  to = "https://{neon-auth-endpoint}/neondb/auth/:splat"
  status = 200
  force = true
```

The Neon Auth endpoint is provided when you enable auth in your Neon project.

### Client Setup

**Astro (server-side):**

```typescript
// src/lib/auth.ts
import { createAuthClient } from '@neondatabase/neon-js/auth';
import type { AuthClient } from '@neondatabase/neon-js/auth';

export function getOrigin(request: Request): string {
  const url = new URL(request.url);
  const proto = request.headers.get('x-forwarded-proto') || url.protocol.replace(':', '');
  return `${proto}://${url.host}`;
}

export function createAuthClientForOrigin(origin: string): AuthClient {
  return createAuthClient(`${origin}/neon-auth`);
}
```

**Vite + React (client-side):**

```typescript
// src/lib/auth.ts
import { createAuthClient } from '@neondatabase/neon-js/auth';

const getAuthUrl = () => {
  if (import.meta.env.PROD) {
    return `${window.location.origin}/neon-auth`;
  }
  return import.meta.env.VITE_NEON_AUTH_URL;
};

export const authClient = createAuthClient(getAuthUrl());
```

---

## Approved Users Safelist

### Database Schema

```typescript
// db/schema.ts
import { boolean, integer, pgTable, varchar, text, timestamp } from 'drizzle-orm/pg-core';

export const approvedUsers = pgTable('approved_users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  email: varchar({ length: 255 }).notNull().unique(),
  name: varchar({ length: 255 }),
  customImage: varchar('custom_image', { length: 255 }),
  isAdmin: boolean('is_admin').notNull().default(false),
  addedAt: timestamp('added_at').defaultNow(),
  addedBy: varchar('added_by', { length: 255 }),
  notes: text(),
});
```

### Auth Utilities

```typescript
// src/lib/auth.ts
import { eq } from 'drizzle-orm';

export type User = {
  id: string;
  email: string;
  name: string | null;
  image: string | null;
};

export async function getUser(request: Request): Promise<User | null> {
  const origin = getOrigin(request);
  const cookies = request.headers.get('cookie') || '';

  try {
    const response = await fetch(`${origin}/neon-auth/get-session`, {
      method: 'GET',
      headers: { cookie: cookies },
    });

    if (!response.ok) return null;

    const data = await response.json();
    if (!data?.user) return null;

    return {
      id: data.user.id,
      email: data.user.email,
      name: data.user.name ?? null,
      image: data.user.image ?? null,
    };
  } catch (error) {
    console.error('Error getting session:', error);
    return null;
  }
}

export async function isUserApproved(email: string): Promise<boolean> {
  const { db, approvedUsers } = await import('../db');

  const [user] = await db
    .select()
    .from(approvedUsers)
    .where(eq(approvedUsers.email, email))
    .limit(1);

  return !!user;
}

export async function isUserAdmin(email: string): Promise<boolean> {
  const { db, approvedUsers } = await import('../db');

  const [user] = await db
    .select()
    .from(approvedUsers)
    .where(eq(approvedUsers.email, email))
    .limit(1);

  return user?.isAdmin ?? false;
}

export async function getUserWithApproval(
  request: Request,
): Promise<{ user: User; isApproved: boolean; isAdmin: boolean } | null> {
  const user = await getUser(request);
  if (!user) return null;

  const { db, approvedUsers } = await import('../db');

  const [approvedUser] = await db
    .select()
    .from(approvedUsers)
    .where(eq(approvedUsers.email, user.email))
    .limit(1);

  return {
    user,
    isApproved: !!approvedUser,
    isAdmin: approvedUser?.isAdmin ?? false,
  };
}
```

---

## Astro Auth Patterns

### Protected Page

```astro
---
// src/pages/dashboard.astro
import Layout from "../layouts/Layout.astro";
import { DashboardPage } from "../components/pages/DashboardPage";
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
<Layout title="Dashboard">
  <DashboardPage user={user} isAdmin={isAdmin} />
</Layout>
```

### Login Page

```astro
---
// src/pages/login.astro
import Layout from "../layouts/Layout.astro";
import { LoginPage } from "../components/pages/LoginPage";
import { getUser } from "../lib/auth";

const user = await getUser(Astro.request);
if (user) {
  return Astro.redirect("/");
}
---
<Layout title="Sign In">
  <LoginPage />
</Layout>
```

### OAuth Callback Handler

```typescript
// src/pages/api/auth/callback.ts
import type { APIRoute } from 'astro';
import { getOrigin } from '../../../lib/auth';
import { logger } from '../../../lib/logger';

const log = logger.scope('CALLBACK');

export const GET: APIRoute = async ({ request, redirect }) => {
  const url = new URL(request.url);
  const verifier = url.searchParams.get('neon_auth_session_verifier');
  const destination = url.searchParams.get('redirect') || '/';
  const origin = getOrigin(request);

  if (!verifier) {
    log.warn('No verifier in callback');
    return redirect('/login', 302);
  }

  try {
    const cookies = request.headers.get('cookie') || '';

    const sessionResponse = await fetch(
      `${origin}/neon-auth/get-session?neon_auth_session_verifier=${verifier}`,
      {
        method: 'GET',
        headers: { cookie: cookies, Origin: origin },
      },
    );

    if (!sessionResponse.ok) {
      log.error('Session error');
      return redirect('/login?message=auth_error', 302);
    }

    const response = redirect(`${destination}?message=signed_in`, 302);

    // Forward session cookies
    for (const cookie of sessionResponse.headers.getSetCookie()) {
      response.headers.append('Set-Cookie', cookie);
    }

    log.info('Session established');
    return response;
  } catch (error) {
    log.error('Callback error:', error);
    return redirect('/login?message=auth_error', 302);
  }
};
```

### Sign In/Out API Routes

```typescript
// src/pages/api/auth/signin.ts
import type { APIRoute } from 'astro';
import { createAuthClientForOrigin, getOrigin } from '../../../lib/auth';

export const GET: APIRoute = async ({ request, redirect }) => {
  const url = new URL(request.url);
  const destination = url.searchParams.get('redirect') || '/';
  const origin = getOrigin(request);

  const authClient = createAuthClientForOrigin(origin);
  const authUrl = await authClient.getOAuthUrl({
    provider: 'google',
    redirectUrl: `${origin}/api/auth/callback?redirect=${encodeURIComponent(destination)}`,
  });

  return redirect(authUrl, 302);
};
```

```typescript
// src/pages/api/auth/signout.ts
import type { APIRoute } from 'astro';
import { createAuthClientForOrigin, getOrigin } from '../../../lib/auth';

export const GET: APIRoute = async ({ request, redirect }) => {
  const origin = getOrigin(request);
  const authClient = createAuthClientForOrigin(origin);

  const response = redirect('/?message=signed_out', 302);

  // Clear session cookies
  const signOutResponse = await authClient.signOut();
  for (const cookie of signOutResponse.headers.getSetCookie()) {
    response.headers.append('Set-Cookie', cookie);
  }

  return response;
};
```

---

## Vite + React Auth Patterns

### Auth Hook

```typescript
// src/hooks/useAuth.ts
import { useState, useEffect, createContext, useContext } from 'react';
import { authClient } from '../lib/auth';

interface AuthContextType {
  user: User | null;
  loading: boolean;
  authenticated: boolean;
  permissions: Permissions;
  signIn: () => void;
  signOut: () => void;
  refetch: () => Promise<void>;
}

export const AuthContext = createContext<AuthContextType | null>(null);

export function useAuthProvider(): AuthContextType {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  const fetchSession = async () => {
    try {
      const session = await authClient.getSession();
      setUser(session?.user ?? null);
    } catch {
      setUser(null);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchSession();
  }, []);

  const signIn = () => {
    authClient.signIn({ provider: 'google' });
  };

  const signOut = async () => {
    await authClient.signOut();
    setUser(null);
  };

  return {
    user,
    loading,
    authenticated: !!user,
    permissions: getPermissions(user),
    signIn,
    signOut,
    refetch: fetchSession,
  };
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

### Protected Routes

```tsx
// src/App.tsx
function RequireAuth({ children }: { children: React.ReactNode }) {
  const { authenticated, loading } = useAuth();

  if (loading) return <PageLoader />;
  if (!authenticated) return <SignInPage />;

  return <>{children}</>;
}

function RequireAdmin({ children }: { children: React.ReactNode }) {
  const { permissions } = useAuth();

  if (!permissions.admin) {
    return <UnauthorizedPage />;
  }

  return <>{children}</>;
}
```

### Header Auth in Netlify Functions

For Vite + React, auth info is passed via headers from the client:

```typescript
// netlify/functions/_shared/auth.ts
export async function requireAuth(req: Request): Promise<AuthResult> {
  const userId = req.headers.get('x-user-id');
  const email = req.headers.get('x-user-email');

  if (!userId || !email) {
    return { authenticated: false };
  }

  const permissions = getPermissionsFromEmail(email);
  return { authenticated: true, userId, email, permissions };
}
```

---

## Cookie Handling (Safari/Localhost)

Neon Auth cookies may need adjustments for Safari ITP and localhost:

```typescript
function fixCookieForLocalhost(cookie: string): string {
  return cookie
    .replace(/^__Secure-/i, '')
    .replace(/;\s*Secure/gi, '')
    .replace(/;\s*Partitioned/gi, '')
    .replace(/;\s*SameSite=None/gi, '; SameSite=Lax');
}

function fixCookieForSafari(cookie: string): string {
  return cookie.replace(/;\s*Partitioned/gi, '').replace(/;\s*SameSite=None/gi, '; SameSite=Lax');
}
```

---

## Common Pitfalls

1. **Forgetting the redirect proxy** - Neon Auth needs the `/neon-auth/*` redirect in netlify.toml
2. **Cookie issues on localhost** - May need to strip `__Secure-` prefix and `Secure` flag
3. **Safari ITP blocking** - Remove `Partitioned` and change `SameSite=None` to `SameSite=Lax`
4. **Not checking approval status** - User being authenticated doesn't mean they're approved
5. **Skipping auth on API routes** - Always verify auth in mutation endpoints
6. **Exposing user data** - Only return necessary fields, never passwords/tokens

---

## Logging Auth Activity

Always log auth events for security monitoring:

```typescript
const log = logger.scope('AUTH');

log.info('User authenticated:', user.email);
log.warn('Unapproved user attempted access:', email);
log.error('Session verification failed');
```

See `logging-and-monitoring` skill for Discord notification integration.

---

## Related Skills

- `astro-best-practices` - Page-level auth patterns
- `vite-best-practices` - Client-side auth hooks
- `logging-and-monitoring` - Auth activity logging
- `data-storage` - approved_users table schema
