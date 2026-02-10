---
name: routing-design
description: URL structure and routing conventions for both Astro and Vite + React projects. Use when designing URL schemes, implementing Rails-style CRUD routes, or working with query parameters. Covers UI routes, API routes, dynamic segments, and the principle that every meaningful view change must have a URL.
---

# Routing Design

## Core Principle

**Every meaningful view change must produce a corresponding URL change.**

Users should be able to:

- Return to any view by bookmarking or sharing the URL
- Use browser back/forward to navigate
- Refresh the page without losing their place

---

## Rails-Style CRUD Routes

### UI Routes

| Route             | Purpose              |
| ----------------- | -------------------- |
| `/posts`          | List all posts       |
| `/posts/new`      | Create new post form |
| `/posts/:id`      | View single post     |
| `/posts/:id/edit` | Edit post form       |

### API Routes

| Method | Route            | Purpose           |
| ------ | ---------------- | ----------------- |
| GET    | `/api/posts`     | List posts (JSON) |
| POST   | `/api/posts`     | Create post       |
| GET    | `/api/posts/:id` | Get single post   |
| PUT    | `/api/posts/:id` | Update post       |
| DELETE | `/api/posts/:id` | Delete post       |

---

## Astro Routing

Routes are file-based in `src/pages/`:

```
src/pages/
├── index.astro           → /
├── posts/
│   ├── index.astro       → /posts
│   ├── new.astro         → /posts/new
│   └── [id]/
│       ├── index.astro   → /posts/:id
│       └── edit.astro    → /posts/:id/edit
└── api/
    └── posts/
        ├── index.ts      → /api/posts (GET, POST)
        └── [id].ts       → /api/posts/:id (GET, PUT, DELETE)
```

### Dynamic Segments

```astro
---
// src/pages/posts/[id]/index.astro
const { id } = Astro.params;
const post = await getPostById(id);

if (!post) {
  return Astro.redirect("/404");
}
---
```

### API Route Handlers

```typescript
// src/pages/api/posts/[id].ts
import type { APIRoute } from 'astro';

export const GET: APIRoute = async ({ params, redirect }) => {
  const post = await getPostById(params.id);
  if (!post) {
    return new Response('Not found', { status: 404 });
  }
  // For Astro SSR, typically redirect or render, not JSON
  return redirect(`/posts/${params.id}`, 302);
};

export const PUT: APIRoute = async ({ params, request, redirect }) => {
  const formData = await request.formData();
  await updatePost(params.id, formData);
  return redirect(`/posts/${params.id}?message=post_updated`, 302);
};

export const DELETE: APIRoute = async ({ params, redirect }) => {
  await deletePost(params.id);
  return redirect('/posts?message=post_deleted', 302);
};
```

---

## Vite + React Routing

Use React Router with explicit route definitions:

```tsx
// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/posts" element={<PostsPage />} />
        <Route path="/posts/new" element={<NewPostPage />} />
        <Route path="/posts/:id" element={<PostPage />} />
        <Route path="/posts/:id/edit" element={<EditPostPage />} />
        <Route path="*" element={<NotFoundPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### Accessing Route Params

```tsx
import { useParams } from 'react-router-dom';

function PostPage() {
  const { id } = useParams<{ id: string }>();
  // Fetch and display post
}
```

### Programmatic Navigation

```tsx
import { useNavigate } from 'react-router-dom';

function CreatePostForm() {
  const navigate = useNavigate();

  const handleSubmit = async () => {
    const newPost = await createPost(data);
    navigate(`/posts/${newPost.id}`);
  };
}
```

---

## Query Parameters

### When to Use Query Parameters

**Use for:**

- Filtering/sorting on list views where permutations are unpredictable
- Feedback messages in Astro SSR (see `feedback` skill)
- Optional view modifiers (e.g., `?view=grid`)

**Avoid for:**

- Core navigation (use path segments instead)
- State that defines the primary content being viewed

### Astro: Feedback Messages

```typescript
// API route returns redirect with message key
return redirect('/posts?message=post_created', 302);
```

```astro
---
// Page reads message key
const message = Astro.url.searchParams.get("message");
---
<Toast messageKey={message} client:load />
```

### React: Filtering

```tsx
import { useSearchParams } from 'react-router-dom';

function PostsPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get('category');
  const sort = searchParams.get('sort') || 'newest';

  const handleCategoryChange = (cat: string) => {
    setSearchParams({ category: cat, sort });
  };

  // Filter posts by category, sort by sort param
}
```

---

## Nested Routes

### Astro: Folder Nesting

```
src/pages/
├── admin/
│   ├── index.astro       → /admin
│   ├── users/
│   │   ├── index.astro   → /admin/users
│   │   └── [id].astro    → /admin/users/:id
│   └── settings.astro    → /admin/settings
```

### React Router: Nested Layouts

```tsx
<Routes>
  <Route element={<Layout />}>
    <Route path="/" element={<HomePage />} />
    <Route path="/posts" element={<PostsPage />} />
  </Route>

  <Route element={<AdminLayout />}>
    <Route path="/admin" element={<AdminDashboard />} />
    <Route path="/admin/users" element={<AdminUsersPage />} />
  </Route>
</Routes>
```

The `<Outlet />` in layout components renders child routes:

```tsx
function Layout() {
  return (
    <>
      <Header />
      <main>
        <Outlet />
      </main>
      <Footer />
    </>
  );
}
```

---

## Modal Routes

Modals should have their own URLs so users can link to them directly.

### Pattern: URL-Driven Modal

```tsx
function PostsPage() {
  const [searchParams] = useSearchParams();
  const editId = searchParams.get('edit');

  return (
    <>
      <PostsList />
      {editId && <EditPostModal postId={editId} />}
    </>
  );
}
```

```tsx
// Opening the modal
<Link to={`/posts?edit=${post.id}`}>Edit</Link>;

// Closing the modal
const [, setSearchParams] = useSearchParams();
const closeModal = () => {
  setSearchParams({});
};
```

---

## API Routes

### Astro API Routes

**Always return redirects, never render directly:**

```typescript
// Good: Returns redirect
return redirect('/posts?message=created', 302);

// Bad: Returns JSON (in Astro SSR context)
return Response.json({ success: true });
```

### Vite + React API Routes (Netlify Functions)

**Return JSON for client consumption:**

```typescript
// Good: Returns JSON
return Response.json({ id: newPost.id, title: newPost.title });

// The client handles navigation
const post = await api.post('/posts', data);
navigate(`/posts/${post.id}`);
```

---

## Anti-Patterns

- **SPA with unchanging URL** - If the view changes, the URL should change
- **Putting primary content identifiers in query params** - Use path segments: `/posts/123` not `/posts?id=123`
- **Deep nesting with no purpose** - Keep URLs shallow: `/admin/users/123` not `/dashboard/admin/section/users/profile/123`
- **Returning JSON from Astro API routes** - Return redirects; pages handle rendering
- **Ignoring browser back button** - Test that navigation history works correctly

---

## Related Skills

- `astro-best-practices` - Astro page patterns
- `vite-best-practices` - React Router setup
- `feedback` - Query param feedback messages
