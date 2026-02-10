---
name: feedback
description: User-facing feedback patterns for actions and errors in both Astro and Vite + React projects. Use when implementing toast notifications, success/error messages, or the query parameter feedback pattern. Covers the message constants approach, Toast component implementation, and display timing rules.
---

# Feedback

## Core Principle

**The UI should always provide the minimum feedback needed for the user to understand what happened.**

Every form submission, action, and error must produce visible feedback.

---

## Two Patterns by Framework

### Astro SSR: Query Parameter Pattern

1. API endpoint performs action
2. Returns redirect with `?message=<key>` query parameter
3. Page renders Toast component that looks up the key
4. Toast displays human-readable message

### Vite + React: JSON Response Pattern

1. Client submits via fetch
2. API returns JSON with result
3. Client calls toast function with message
4. Toast displays the message

---

## Astro Pattern Implementation

### Message Constants

```typescript
// src/lib/messages.ts
export const MESSAGES = {
  // Success messages
  entry_created: { type: 'success', text: 'Your entry has been submitted!' },
  entry_updated: { type: 'success', text: 'Your entry has been updated.' },
  entry_deleted: { type: 'success', text: 'Your entry has been deleted.' },
  profile_updated: { type: 'success', text: 'Your profile has been updated.' },
  signed_in: { type: 'success', text: "Welcome! You've signed in successfully." },
  signed_out: { type: 'success', text: "You've been signed out." },

  // Error messages
  entry_exists: { type: 'error', text: 'You already have an entry.' },
  auth_error: { type: 'error', text: 'Authentication failed. Please try again.' },
  unauthorized: { type: 'error', text: "You don't have permission to do that." },
  validation_error: { type: 'error', text: 'Please check your input and try again.' },
  create_failed: { type: 'error', text: 'Failed to create. Please try again.' },
  update_failed: { type: 'error', text: 'Failed to update. Please try again.' },
  delete_failed: { type: 'error', text: 'Failed to delete. Please try again.' },
} as const;

export type MessageKey = keyof typeof MESSAGES;
export type MessageType = (typeof MESSAGES)[MessageKey]['type'];
```

### API Route Returns Redirect

```typescript
// src/pages/api/entries.ts
export const POST: APIRoute = async ({ request, redirect }) => {
  try {
    await createEntry(data);
    return redirect('/entries?message=entry_created', 302);
  } catch (error) {
    return redirect('/entries?message=create_failed', 302);
  }
};
```

### Toast Component (React with client directive)

```tsx
// src/components/ui/Toast.tsx
import { useState, useEffect } from 'react';
import { CheckCircle, XCircle, X } from 'lucide-react';
import { MESSAGES, type MessageKey } from '../../lib/messages';

interface ToastProps {
  messageKey?: string;
}

export function Toast({ messageKey }: ToastProps) {
  const [visible, setVisible] = useState(false);

  const message = messageKey && messageKey in MESSAGES ? MESSAGES[messageKey as MessageKey] : null;

  useEffect(() => {
    if (!message) return;

    setVisible(true);

    // Clear the query param from URL without reload
    const url = new URL(window.location.href);
    url.searchParams.delete('message');
    window.history.replaceState({}, '', url.toString());

    // Auto-dismiss after 4 seconds
    const timer = setTimeout(() => setVisible(false), 4000);
    return () => clearTimeout(timer);
  }, [message]);

  if (!message || !visible) return null;

  const isSuccess = message.type === 'success';
  const Icon = isSuccess ? CheckCircle : XCircle;

  return (
    <div
      role="alert"
      className={`
        fixed top-4 left-1/2 -translate-x-1/2 z-50 max-w-sm w-[calc(100%-2rem)]
        flex items-center gap-3 px-4 py-3 rounded-lg shadow-xl
        ${
          isSuccess
            ? 'bg-green-900/90 border border-green-700 text-green-100'
            : 'bg-red-900/90 border border-red-700 text-red-100'
        }
      `}
    >
      <Icon className="w-5 h-5 flex-shrink-0" />
      <p className="text-sm font-medium">{message.text}</p>
      <button
        onClick={() => setVisible(false)}
        className="ml-auto p-1 rounded hover:bg-white/10"
        aria-label="Dismiss"
      >
        <X className="w-4 h-4" />
      </button>
    </div>
  );
}
```

### Usage in Layout

```astro
---
// src/layouts/Layout.astro
import { Toast } from "../components/ui/Toast";

const messageKey = Astro.url.searchParams.get("message") || undefined;
---
<html>
  <body>
    <Toast messageKey={messageKey} client:load />
    <slot />
  </body>
</html>
```

---

## Vite + React Pattern Implementation

### Toast Hook

```tsx
// src/components/Toast.tsx
import { useState, useCallback, createContext, useContext } from 'react';
import { CheckCircle, XCircle, X } from 'lucide-react';

type ToastType = 'success' | 'error';

interface Toast {
  id: number;
  type: ToastType;
  message: string;
}

interface ToastContextType {
  showToast: (type: ToastType, message: string) => void;
}

export const ToastContext = createContext<ToastContextType | null>(null);

let toastId = 0;

export function useToastProvider() {
  const [toasts, setToasts] = useState<Toast[]>([]);

  const showToast = useCallback((type: ToastType, message: string) => {
    const id = ++toastId;
    setToasts((prev) => [...prev, { id, type, message }]);

    // Auto-dismiss after 6 seconds
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id));
    }, 6000);
  }, []);

  const dismissToast = useCallback((id: number) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }, []);

  return { toasts, showToast, dismissToast };
}

export function useToast() {
  const context = useContext(ToastContext);
  if (!context) throw new Error('useToast must be used within ToastProvider');
  return context;
}

export function ToastContainer({
  toasts,
  onDismiss,
}: {
  toasts: Toast[];
  onDismiss: (id: number) => void;
}) {
  if (toasts.length === 0) return null;

  return (
    <div className="fixed bottom-4 right-4 z-50 flex flex-col gap-2">
      {toasts.map((toast) => (
        <ToastItem key={toast.id} toast={toast} onDismiss={onDismiss} />
      ))}
    </div>
  );
}

function ToastItem({ toast, onDismiss }: { toast: Toast; onDismiss: (id: number) => void }) {
  const isSuccess = toast.type === 'success';
  const Icon = isSuccess ? CheckCircle : XCircle;

  return (
    <div
      className={`
      flex items-center gap-3 px-4 py-3 rounded-lg shadow-lg border
      ${
        isSuccess
          ? 'bg-green-900/90 border-green-700 text-green-100'
          : 'bg-red-900/90 border-red-700 text-red-100'
      }
    `}
    >
      <Icon className="w-5 h-5 flex-shrink-0" />
      <span className="text-sm font-medium">{toast.message}</span>
      <button onClick={() => onDismiss(toast.id)} className="ml-2 p-1 rounded hover:bg-white/10">
        <X className="w-4 h-4" />
      </button>
    </div>
  );
}
```

### Usage in App

```tsx
// src/App.tsx
function App() {
  const toastValue = useToastProvider();

  return (
    <ToastContext.Provider value={{ showToast: toastValue.showToast }}>
      <BrowserRouter>
        <ToastContainer toasts={toastValue.toasts} onDismiss={toastValue.dismissToast} />
        <Routes>...</Routes>
      </BrowserRouter>
    </ToastContext.Provider>
  );
}
```

### Usage in Components

```tsx
function CreateForm() {
  const { showToast } = useToast();

  const handleSubmit = async (data: FormData) => {
    try {
      await api.post('/items', data);
      showToast('success', 'Item created successfully');
    } catch (error) {
      showToast('error', error.message || 'Failed to create item');
    }
  };
}
```

---

## Display Rules

| Rule                         | Implementation                     |
| ---------------------------- | ---------------------------------- |
| Visible regardless of scroll | Use `fixed` positioning            |
| Stays visible ~4-6 seconds   | `setTimeout` for auto-dismiss      |
| User can close manually      | Close button with click handler    |
| User can copy message        | Text is selectable (not prevented) |
| Multiple toasts stack        | Array of toasts, flex column       |

---

## Positioning

**Astro (full-page navigations):** Top center - user's attention is at the top after navigation

**Vite + React (SPA):** Bottom right - less intrusive during in-page interactions

---

## Error Message Guidelines

- Be specific but not technical
- Suggest what to do next
- Don't expose internal error details
- Log full errors server-side

```typescript
// Good
'Failed to save. Please try again.';
"You don't have permission to do that.";
'Please check your input and try again.';

// Bad
'Error: FOREIGN_KEY_CONSTRAINT_VIOLATION';
'null is not an object';
'500 Internal Server Error';
```

---

## Pre-Rendered Pages with Forms

For forms on static pages (e.g., contact form on marketing site), consider Astro Server Islands:

```astro
<ContactForm server:defer />
```

The form renders and processes on the server, can show feedback without navigating away.

---

## Related Skills

- `forms` - Form submission patterns
- `astro-best-practices` - Astro API route patterns
- `vite-best-practices` - React patterns
- `logging-and-monitoring` - Error logging
