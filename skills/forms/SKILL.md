---
name: forms
description: Form implementation patterns for both Astro and Vite + React projects. Use when building form components, handling submissions, implementing validation, or integrating with the feedback system. Covers standard HTTP forms (Astro) and client-side JSON forms (React).
---

# Forms

Form implementation patterns across both frameworks.

---

## Two Patterns by Framework

### Astro SSR: Standard HTTP Forms

- Forms submit via native browser behavior (no JS required)
- API endpoint processes form, returns redirect
- Redirect includes feedback message key
- Works with progressive enhancement

### Vite + React: Client-Side JSON Forms

- Forms submit via fetch with JSON body
- API returns JSON response
- Client displays feedback via toast
- Better UX for complex interactions

---

## Astro Form Pattern

### Basic Form Component

```tsx
// src/components/entries/EntryForm.tsx
interface EntryFormProps {
  entry?: Entry;
  action?: string;
}

export function EntryForm({ entry, action = "/api/entries" }: EntryFormProps) {
  const isEdit = !!entry;

  return (
    <form action={action} method="post">
      {/* Method override for PUT */}
      {isEdit && <input type="hidden" name="_method" value="PUT" />}
      {isEdit && <input type="hidden" name="id" value={entry.id} />}

      <div>
        <label htmlFor="title">Title</label>
        <input
          type="text"
          id="title"
          name="title"
          defaultValue={entry?.title}
          required
          maxLength={255}
        />
      </div>

      <div>
        <label htmlFor="description">Description</label>
        <textarea
          id="description"
          name="description"
          defaultValue={entry?.description}
          required
          rows={4}
        />
      </div>

      <div>
        <label>
          <input
            type="checkbox"
            name="needsPower"
            defaultChecked={entry?.needsPower}
          />
          Needs power outlet
        </label>
      </div>

      <button type="submit">
        {isEdit ? "Update Entry" : "Create Entry"}
      </button>
    </form>
  );
}
```

### API Route Handler

```typescript
// src/pages/api/entries.ts
import type { APIRoute } from "astro";
import { getUserWithApproval } from "../../lib/auth";
import { createEntry, updateEntry, deleteEntry } from "../../lib/entries";
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
  const method = formData.get("_method")?.toString() || "POST";

  // Route to appropriate handler
  if (method === "DELETE") {
    return handleDelete(formData, auth.user.email, redirect);
  }
  if (method === "PUT") {
    return handleUpdate(formData, auth.user.email, redirect);
  }
  return handleCreate(formData, auth.user, redirect);
};

async function handleCreate(
  formData: FormData,
  user: { email: string; name: string | null },
  redirect: (path: string, status?: number) => Response
) {
  const title = formData.get("title")?.toString().trim();
  const description = formData.get("description")?.toString().trim();
  const needsPower = formData.get("needsPower") === "on";

  // Validation
  if (!title || !description) {
    log.warn("Missing required fields");
    return redirect("/my-entry?message=validation_error", 302);
  }

  try {
    await createEntry({
      title,
      description,
      needsPower,
      userEmail: user.email,
      userName: user.name,
    });
    log.info("Entry created for:", user.email);
    return redirect("/my-entry?message=entry_created", 302);
  } catch (error) {
    log.error("Failed to create entry:", error);
    return redirect("/my-entry?message=create_failed", 302);
  }
}

async function handleUpdate(
  formData: FormData,
  userEmail: string,
  redirect: (path: string, status?: number) => Response
) {
  const id = parseInt(formData.get("id")?.toString() || "0", 10);
  const title = formData.get("title")?.toString().trim();
  const description = formData.get("description")?.toString().trim();
  const needsPower = formData.get("needsPower") === "on";

  if (!id || !title || !description) {
    return redirect("/my-entry/edit?message=validation_error", 302);
  }

  try {
    await updateEntry(id, userEmail, { title, description, needsPower });
    log.info("Entry updated for:", userEmail);
    return redirect("/my-entry?message=entry_updated", 302);
  } catch (error) {
    log.error("Failed to update entry:", error);
    return redirect("/my-entry/edit?message=update_failed", 302);
  }
}

async function handleDelete(
  formData: FormData,
  userEmail: string,
  redirect: (path: string, status?: number) => Response
) {
  const id = parseInt(formData.get("id")?.toString() || "0", 10);

  if (!id) {
    return redirect("/my-entry?message=validation_error", 302);
  }

  try {
    await deleteEntry(id, userEmail);
    log.info("Entry deleted for:", userEmail);
    return redirect("/?message=entry_deleted", 302);
  } catch (error) {
    log.error("Failed to delete entry:", error);
    return redirect("/my-entry?message=delete_failed", 302);
  }
}
```

### Delete Form (Inline)

```tsx
<form action="/api/entries" method="post">
  <input type="hidden" name="_method" value="DELETE" />
  <input type="hidden" name="id" value={entry.id} />
  <button type="submit" className="text-red-500">
    Delete
  </button>
</form>
```

---

## Vite + React Form Pattern

### Basic Form Component

```tsx
// src/components/ItemForm.tsx
import { useState } from "react";
import { useToast } from "./Toast";
import { api } from "../lib/api";

interface ItemFormProps {
  item?: Item;
  onSuccess: () => void;
  onCancel?: () => void;
}

export function ItemForm({ item, onSuccess, onCancel }: ItemFormProps) {
  const [submitting, setSubmitting] = useState(false);
  const [errors, setErrors] = useState<Record<string, string>>({});
  const { showToast } = useToast();

  const isEdit = !!item;

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setErrors({});
    setSubmitting(true);

    const formData = new FormData(e.currentTarget);
    const data = {
      title: formData.get("title")?.toString().trim(),
      description: formData.get("description")?.toString().trim(),
      priority: formData.get("priority")?.toString(),
    };

    // Client-side validation
    const validationErrors: Record<string, string> = {};
    if (!data.title) validationErrors.title = "Title is required";
    if (!data.description) validationErrors.description = "Description is required";

    if (Object.keys(validationErrors).length > 0) {
      setErrors(validationErrors);
      setSubmitting(false);
      return;
    }

    try {
      if (isEdit) {
        await api.put(`/items/${item.id}`, data);
        showToast("success", "Item updated successfully");
      } else {
        await api.post("/items", data);
        showToast("success", "Item created successfully");
      }
      onSuccess();
    } catch (error) {
      const message = error instanceof Error ? error.message : "Operation failed";
      showToast("error", message);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label htmlFor="title" className="block text-sm font-medium">
          Title
        </label>
        <input
          type="text"
          id="title"
          name="title"
          defaultValue={item?.title}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.title && (
          <p className="mt-1 text-sm text-red-500">{errors.title}</p>
        )}
      </div>

      <div>
        <label htmlFor="description" className="block text-sm font-medium">
          Description
        </label>
        <textarea
          id="description"
          name="description"
          defaultValue={item?.description}
          rows={4}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.description && (
          <p className="mt-1 text-sm text-red-500">{errors.description}</p>
        )}
      </div>

      <div>
        <label htmlFor="priority" className="block text-sm font-medium">
          Priority
        </label>
        <select
          id="priority"
          name="priority"
          defaultValue={item?.priority || "medium"}
          className="mt-1 block w-full rounded border p-2"
        >
          <option value="low">Low</option>
          <option value="medium">Medium</option>
          <option value="high">High</option>
        </select>
      </div>

      <div className="flex gap-2">
        <button
          type="submit"
          disabled={submitting}
          className="px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-50"
        >
          {submitting ? "Saving..." : isEdit ? "Update" : "Create"}
        </button>
        {onCancel && (
          <button
            type="button"
            onClick={onCancel}
            className="px-4 py-2 border rounded"
          >
            Cancel
          </button>
        )}
      </div>
    </form>
  );
}
```

### Delete Button

```tsx
function DeleteButton({ itemId, onSuccess }: { itemId: number; onSuccess: () => void }) {
  const [deleting, setDeleting] = useState(false);
  const { showToast } = useToast();

  const handleDelete = async () => {
    if (!confirm("Are you sure you want to delete this item?")) return;

    setDeleting(true);
    try {
      await api.delete(`/items/${itemId}`);
      showToast("success", "Item deleted");
      onSuccess();
    } catch (error) {
      showToast("error", "Failed to delete item");
    } finally {
      setDeleting(false);
    }
  };

  return (
    <button
      onClick={handleDelete}
      disabled={deleting}
      className="text-red-500 disabled:opacity-50"
    >
      {deleting ? "Deleting..." : "Delete"}
    </button>
  );
}
```

---

## File Upload Forms

### Astro: Multipart Form

```astro
<form action="/api/upload" method="post" enctype="multipart/form-data">
  <input
    type="file"
    name="image"
    accept="image/jpeg,image/png,image/webp"
    required
  />
  <button type="submit">Upload</button>
</form>
```

### Vite + React: FormData Upload

```tsx
const handleUpload = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  const formData = new FormData(e.currentTarget);

  const response = await fetch("/api/upload", {
    method: "POST",
    body: formData, // Don't set Content-Type; browser sets it with boundary
  });

  if (!response.ok) {
    const error = await response.json();
    showToast("error", error.message);
    return;
  }

  showToast("success", "File uploaded");
};
```

---

## Validation

### Client-Side (React)

```tsx
const validationErrors: Record<string, string> = {};

if (!data.title) {
  validationErrors.title = "Title is required";
} else if (data.title.length > 255) {
  validationErrors.title = "Title must be less than 255 characters";
}

if (!data.email) {
  validationErrors.email = "Email is required";
} else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
  validationErrors.email = "Invalid email format";
}
```

### Server-Side (Always Required)

```typescript
// Never trust client validation
const title = formData.get("title")?.toString().trim();
if (!title) {
  return redirect("/form?message=validation_error", 302);
}
if (title.length > 255) {
  return redirect("/form?message=title_too_long", 302);
}
```

---

## Reusable Input Components

```tsx
// src/components/Input.tsx
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

export function Input({ label, error, id, ...props }: InputProps) {
  return (
    <div>
      <label htmlFor={id} className="block text-sm font-medium">
        {label}
      </label>
      <input
        id={id}
        className={`mt-1 block w-full rounded border p-2 ${
          error ? "border-red-500" : "border-gray-300"
        }`}
        {...props}
      />
      {error && <p className="mt-1 text-sm text-red-500">{error}</p>}
    </div>
  );
}
```

---

## Netlify Forms (Not Used)

Most projects have a database, making Netlify Forms unnecessary. If you need simple form collection without a database, Netlify Forms is an option, but it's outside these conventions.

---

## Anti-Patterns

- **Skipping server validation** - Client validation can be bypassed
- **Exposing error details** - Log server-side, show user-friendly message
- **Not disabling submit during processing** - Prevents double submission
- **Forgetting method override** - HTML forms only support GET/POST

---

## Related Skills

- `feedback` - Toast and message patterns
- `astro-best-practices` - API route patterns
- `vite-best-practices` - React patterns
- `netlify-functions` - API endpoint implementation
