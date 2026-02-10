---
name: ui-design
description: UI conventions, component libraries, and design system patterns for Sean's projects. Use when implementing visual designs, creating consistent UI patterns, or building accessible interfaces. Covers Tailwind CSS usage, component patterns, responsive design, and accessibility baseline.
---

# UI Design

UI conventions and design system patterns.

---

## Core Approach

Sean's projects use **Tailwind CSS v4** for styling. Focus on:

1. **Functional over decorative** - UI serves the task
2. **Consistent patterns** - Reuse components, not copy-paste
3. **Progressive rendering** - Show something immediately
4. **Accessibility by default** - Semantic HTML, keyboard nav, ARIA when needed

---

## Tailwind CSS v4 Setup

### Vite + React

```typescript
// vite.config.ts
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

```css
/* src/index.css */
@import "tailwindcss";
```

### Astro

```javascript
// astro.config.mjs
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  vite: {
    plugins: [tailwindcss()],
  },
});
```

---

## Component Patterns

### Button

```tsx
// src/components/ui/Button.tsx
import { type ButtonHTMLAttributes } from "react";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "danger" | "ghost";
  size?: "sm" | "md" | "lg";
}

export function Button({
  variant = "primary",
  size = "md",
  className = "",
  disabled,
  children,
  ...props
}: ButtonProps) {
  const baseStyles = "inline-flex items-center justify-center font-medium rounded-lg transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed";

  const variants = {
    primary: "bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500",
    secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300 focus:ring-gray-500",
    danger: "bg-red-600 text-white hover:bg-red-700 focus:ring-red-500",
    ghost: "bg-transparent hover:bg-gray-100 focus:ring-gray-500",
  };

  const sizes = {
    sm: "px-3 py-1.5 text-sm",
    md: "px-4 py-2 text-sm",
    lg: "px-6 py-3 text-base",
  };

  return (
    <button
      className={`${baseStyles} ${variants[variant]} ${sizes[size]} ${className}`}
      disabled={disabled}
      {...props}
    >
      {children}
    </button>
  );
}
```

### Input

```tsx
// src/components/ui/Input.tsx
import { type InputHTMLAttributes, forwardRef } from "react";

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, id, className = "", ...props }, ref) => {
    return (
      <div>
        {label && (
          <label htmlFor={id} className="block text-sm font-medium text-gray-700 mb-1">
            {label}
          </label>
        )}
        <input
          ref={ref}
          id={id}
          className={`
            block w-full rounded-lg border px-3 py-2 text-sm
            focus:outline-none focus:ring-2 focus:ring-blue-500
            ${error ? "border-red-500" : "border-gray-300"}
            ${className}
          `}
          {...props}
        />
        {error && (
          <p className="mt-1 text-sm text-red-500">{error}</p>
        )}
      </div>
    );
  }
);
```

### Card

```tsx
// src/components/ui/Card.tsx
interface CardProps {
  children: React.ReactNode;
  className?: string;
}

export function Card({ children, className = "" }: CardProps) {
  return (
    <div className={`bg-white rounded-lg border border-gray-200 shadow-sm ${className}`}>
      {children}
    </div>
  );
}

Card.Header = function CardHeader({ children, className = "" }: CardProps) {
  return (
    <div className={`px-4 py-3 border-b border-gray-200 ${className}`}>
      {children}
    </div>
  );
};

Card.Body = function CardBody({ children, className = "" }: CardProps) {
  return <div className={`p-4 ${className}`}>{children}</div>;
};

Card.Footer = function CardFooter({ children, className = "" }: CardProps) {
  return (
    <div className={`px-4 py-3 border-t border-gray-200 bg-gray-50 rounded-b-lg ${className}`}>
      {children}
    </div>
  );
};
```

---

## Skeleton Loading

```tsx
// src/components/ui/Skeleton.tsx
interface SkeletonProps {
  className?: string;
}

export function Skeleton({ className = "" }: SkeletonProps) {
  return (
    <div className={`animate-pulse bg-gray-200 rounded ${className}`} />
  );
}

export function CardSkeleton() {
  return (
    <div className="bg-white rounded-lg border border-gray-200 p-4">
      <Skeleton className="h-4 w-3/4 mb-3" />
      <Skeleton className="h-3 w-full mb-2" />
      <Skeleton className="h-3 w-2/3" />
    </div>
  );
}

export function TableRowSkeleton({ columns = 4 }: { columns?: number }) {
  return (
    <tr>
      {Array.from({ length: columns }).map((_, i) => (
        <td key={i} className="px-4 py-3">
          <Skeleton className="h-4 w-full" />
        </td>
      ))}
    </tr>
  );
}
```

---

## Layout Patterns

### Page Container

```tsx
export function Container({ children, className = "" }: { children: React.ReactNode; className?: string }) {
  return (
    <div className={`max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 ${className}`}>
      {children}
    </div>
  );
}
```

### Stack Layout

```tsx
export function Stack({
  children,
  gap = "4",
  className = "",
}: {
  children: React.ReactNode;
  gap?: string;
  className?: string;
}) {
  return (
    <div className={`flex flex-col gap-${gap} ${className}`}>
      {children}
    </div>
  );
}
```

---

## Responsive Design

### Mobile-First Breakpoints

```tsx
// Always design mobile-first, then add breakpoints
<div className="
  grid grid-cols-1
  sm:grid-cols-2
  lg:grid-cols-3
  xl:grid-cols-4
  gap-4
">
  {items.map(item => <Card key={item.id} {...item} />)}
</div>
```

### Responsive Typography

```tsx
<h1 className="text-2xl sm:text-3xl lg:text-4xl font-bold">
  Page Title
</h1>
```

### Hide/Show at Breakpoints

```tsx
// Show on mobile, hide on desktop
<div className="block lg:hidden">Mobile Menu</div>

// Hide on mobile, show on desktop
<div className="hidden lg:block">Desktop Sidebar</div>
```

---

## Accessibility Baseline

### Semantic HTML

```tsx
// Good
<button onClick={handleClick}>Submit</button>
<nav>...</nav>
<main>...</main>
<article>...</article>

// Bad
<div onClick={handleClick}>Submit</div>
<div className="nav">...</div>
```

### Focus States

Always visible focus indicators:

```tsx
<button className="focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">
  Click me
</button>
```

### ARIA When Needed

```tsx
// Loading state
<button disabled aria-busy="true">
  <Spinner aria-hidden="true" />
  <span>Loading...</span>
</button>

// Modals
<div role="dialog" aria-modal="true" aria-labelledby="modal-title">
  <h2 id="modal-title">Modal Title</h2>
  ...
</div>

// Alerts
<div role="alert" aria-live="polite">
  Form submitted successfully
</div>
```

### Labels

```tsx
// Always associate labels with inputs
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// Or use aria-label for icon buttons
<button aria-label="Close menu">
  <XIcon />
</button>
```

---

## Color Patterns

### Dark Mode Support

```tsx
// Use dark: prefix for dark mode styles
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
  Content
</div>
```

### Semantic Colors

Define semantic colors in your CSS:

```css
@theme {
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-danger: #ef4444;
  --color-info: #3b82f6;
}
```

---

## Typography

### Font Stack

```css
@theme {
  --font-sans: ui-sans-serif, system-ui, sans-serif;
  --font-mono: ui-monospace, SFMono-Regular, monospace;
}
```

### Type Scale

```tsx
<h1 className="text-3xl font-bold">Page Title</h1>
<h2 className="text-2xl font-semibold">Section Title</h2>
<h3 className="text-xl font-medium">Subsection</h3>
<p className="text-base">Body text</p>
<span className="text-sm text-gray-500">Helper text</span>
```

---

## Icon Usage

Use Lucide React for icons:

```bash
npm install lucide-react
```

```tsx
import { Check, X, ChevronDown, Menu } from "lucide-react";

<Button>
  <Check className="w-4 h-4 mr-2" />
  Confirm
</Button>
```

---

## Anti-Patterns

- **Inline styles** - Use Tailwind classes
- **Inconsistent spacing** - Use Tailwind's spacing scale
- **Missing focus states** - Every interactive element needs visible focus
- **Color-only indicators** - Don't rely solely on color for meaning
- **Fixed widths** - Use responsive utilities
- **Pixel-perfect matching** - Focus on consistency, not exact replication

---

## Related Skills

- `component-design` - Component architecture
- `feedback` - Toast and notification components
- `astro-best-practices` - React components in Astro
- `vite-best-practices` - React component patterns
