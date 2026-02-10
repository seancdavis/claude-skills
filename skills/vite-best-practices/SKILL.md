---
name: vite-best-practices
description: Conventions and patterns for Vite + React projects deployed to Netlify. Use when building React SPAs, configuring the Netlify Vite plugin, setting up React Router, implementing progressive UI rendering, or handling API data fetching. Covers client-side routing, loading states, form handling, and Netlify Functions integration.
---

# Vite + React Best Practices

Conventions and patterns for building Vite + React applications deployed to Netlify.

---

## Core Principles

1. **TypeScript always** - No exceptions
2. **Progressive rendering** - Show UI fast, load data incrementally
3. **URL reflects state** - Every view change updates the URL
4. **API-driven data** - Fetch via Netlify Functions, responses are JSON

---

## Project Configuration

### vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import netlify from '@netlify/vite-plugin'

export default defineConfig({
  plugins: [react(), tailwindcss(), netlify()],
})
```

The Netlify Vite plugin:
- Injects Netlify environment variables into `npm run dev`
- Provides access to Netlify Blobs, DB, and other services locally
- No need for `netlify dev` wrapper

### package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  }
}
```

### netlify.toml

```toml
[build]
  command = "npm run build"
  publish = "dist"
  functions = "netlify/functions"

[dev]
  command = "npm run dev"
  targetPort = 5173
```

---

## React Router Setup

### App.tsx Structure

```tsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { Layout } from "./components/Layout";
import { AdminLayout } from "./components/AdminLayout";
import { HomePage } from "./pages/HomePage";
import { ItemPage } from "./pages/ItemPage";
import { AdminPage } from "./pages/admin/AdminPage";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        {/* Public routes */}
        <Route element={<Layout />}>
          <Route path="/" element={<HomePage />} />
          <Route path="/items" element={<ItemsPage />} />
          <Route path="/items/:id" element={<ItemPage />} />
        </Route>

        {/* Admin routes */}
        <Route element={<AdminLayout />}>
          <Route path="/admin" element={<Navigate to="/admin/dashboard" replace />} />
          <Route path="/admin/dashboard" element={<AdminPage />} />
          <Route path="/admin/items" element={<AdminItemsPage />} />
          <Route path="/admin/items/:id" element={<AdminItemPage />} />
        </Route>

        {/* Catch-all */}
        <Route path="*" element={<Navigate to="/" replace />} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

### URL-Driven Views

Every meaningful view change should produce a URL change:

```tsx
// Good: Modal has its own route
<Route path="/items/:id" element={<ItemPage />} />
<Route path="/items/:id/edit" element={<ItemPage showEditModal />} />

// Good: Use URL params for view state
function ItemPage() {
  const { id } = useParams();
  const [searchParams] = useSearchParams();
  const showEdit = searchParams.get("edit") === "true";
  // ...
}
```

---

## Progressive Rendering Pattern

Render static UI immediately, load dynamic content in stages.

### Pattern: Static Shell → Skeleton → Data

```tsx
function KanbanBoard() {
  const [columns, setColumns] = useState<Column[] | null>(null);

  useEffect(() => {
    fetchColumns().then(setColumns);
  }, []);

  return (
    <div className="kanban-board">
      {/* Static header renders immediately */}
      <header>
        <h1>Project Board</h1>
        <Button>Add Column</Button>
      </header>

      {/* Columns: skeleton → data */}
      <div className="columns">
        {columns === null ? (
          <ColumnsSkeleton count={3} />
        ) : (
          columns.map(col => (
            <KanbanColumn key={col.id} column={col} />
          ))
        )}
      </div>
    </div>
  );
}
```

### Pattern: Load at Smallest Increment

```tsx
function KanbanColumn({ column }: { column: Column }) {
  const [items, setItems] = useState<Item[] | null>(null);

  useEffect(() => {
    fetchColumnItems(column.id).then(setItems);
  }, [column.id]);

  return (
    <div className="column">
      {/* Column header renders immediately */}
      <h2>{column.name}</h2>

      {/* Items load independently */}
      {items === null ? (
        <ItemsSkeleton count={5} />
      ) : (
        items.map(item => <KanbanItem key={item.id} item={item} />)
      )}
    </div>
  );
}
```

### Reusable Loading Components

```tsx
// src/components/LoadingSpinner.tsx
export function LoadingSpinner({ size = "md" }: { size?: "sm" | "md" | "lg" }) {
  const sizes = { sm: "w-4 h-4", md: "w-8 h-8", lg: "w-12 h-12" };
  return (
    <div className={`animate-spin ${sizes[size]} border-2 border-current border-t-transparent rounded-full`} />
  );
}

export function PageLoader({ message }: { message?: string }) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <LoadingSpinner size="lg" />
      {message && <p className="mt-4 text-muted">{message}</p>}
    </div>
  );
}

// Skeleton components
export function CardSkeleton() {
  return <div className="h-24 bg-muted animate-pulse rounded-lg" />;
}

export function ColumnsSkeleton({ count }: { count: number }) {
  return (
    <>
      {Array.from({ length: count }).map((_, i) => (
        <div key={i} className="w-72 bg-muted animate-pulse rounded-lg h-96" />
      ))}
    </>
  );
}
```

---

## Data Fetching

### API Utility

```typescript
// src/lib/api.ts
const API_BASE = "/api";

async function request<T>(path: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...options?.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ error: "Request failed" }));
    throw new Error(error.error || "Request failed");
  }

  return response.json();
}

export const api = {
  get: <T>(path: string) => request<T>(path),
  post: <T>(path: string, data: unknown) =>
    request<T>(path, { method: "POST", body: JSON.stringify(data) }),
  put: <T>(path: string, data: unknown) =>
    request<T>(path, { method: "PUT", body: JSON.stringify(data) }),
  delete: <T>(path: string) =>
    request<T>(path, { method: "DELETE" }),
};
```

### Custom Hooks for Data

```typescript
// src/hooks/useItems.ts
import { useState, useEffect } from "react";
import { api } from "../lib/api";

export function useItems() {
  const [items, setItems] = useState<Item[] | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    api.get<Item[]>("/items")
      .then(setItems)
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  const refetch = () => {
    setLoading(true);
    api.get<Item[]>("/items")
      .then(setItems)
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  };

  return { items, error, loading, refetch };
}
```

---

## Form Handling

Forms submit via client-side JSON requests, display feedback based on response.

```tsx
function CreateItemForm({ onSuccess }: { onSuccess: () => void }) {
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const { showToast } = useToast();

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setError(null);
    setSubmitting(true);

    const formData = new FormData(e.currentTarget);
    const data = {
      title: formData.get("title"),
      description: formData.get("description"),
    };

    try {
      await api.post("/items", data);
      showToast("success", "Item created successfully");
      onSuccess();
    } catch (err) {
      const message = err instanceof Error ? err.message : "Failed to create item";
      setError(message);
      showToast("error", message);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <Input name="title" label="Title" required />
      <TextArea name="description" label="Description" />
      {error && <p className="text-danger">{error}</p>}
      <Button type="submit" disabled={submitting}>
        {submitting ? "Creating..." : "Create Item"}
      </Button>
    </form>
  );
}
```

---

## Folder Structure

```
src/
├── components/
│   ├── Button.tsx
│   ├── Input.tsx
│   ├── Toast.tsx
│   ├── Layout.tsx
│   └── AdminLayout.tsx
├── hooks/
│   ├── useAuth.ts
│   ├── useToast.ts
│   └── use{Feature}.ts
├── lib/
│   ├── api.ts
│   ├── auth.ts
│   └── utils.ts
├── pages/
│   ├── HomePage.tsx
│   ├── admin/
│   │   └── AdminPage.tsx
│   └── {Feature}Page.tsx
├── App.tsx
└── main.tsx
netlify/
└── functions/
    ├── _shared/
    │   ├── auth.ts
    │   ├── db.ts
    │   └── utils.ts
    └── {endpoint}.ts
```

---

## Context Providers

```tsx
// App.tsx
function App() {
  const authValue = useAuthProvider();
  const toastValue = useToastProvider();

  if (authValue.loading) {
    return <PageLoader message="Loading..." />;
  }

  return (
    <AuthContext.Provider value={authValue}>
      <ToastContext.Provider value={toastValue}>
        <BrowserRouter>
          <ToastContainer />
          <Routes>...</Routes>
        </BrowserRouter>
      </ToastContext.Provider>
    </AuthContext.Provider>
  );
}
```

---

## Anti-Patterns

- **Don't build SPAs where the URL never changes** - Use React Router, reflect state in URL
- **Don't render everything at once** - Use progressive loading
- **Don't skip loading states** - Always show skeletons or spinners
- **Don't fetch in components without cleanup** - Handle unmount cases
- **Don't use `netlify dev`** - The Vite plugin handles environment injection

---

## Related Skills

- `netlify-functions` - API endpoint implementation
- `routing-design` - URL structure conventions
- `feedback` - Toast notifications
- `forms` - Form handling patterns
- `auth-design` - Authentication patterns
- `component-design` - Component organization
