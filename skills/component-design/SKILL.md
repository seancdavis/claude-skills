---
name: component-design
description: Component architecture and organization patterns for React components in both Astro and Vite + React projects. Use when organizing component files, implementing loading/skeleton patterns, structuring page components, or making decisions about component granularity. Covers single-responsibility, DRY principles, and progressive rendering.
---

# Component Design

## Core Principles

1. **Single responsibility** - Each component does essentially one thing
2. **DRY** - Reuse components with different props rather than duplicating
3. **Not too granular** - Don't need 10 files to understand a paragraph
4. **Easy to find, easy to understand** - Clear naming and organization
5. **Progressive rendering** - Static shell → skeleton → dynamic content

---

## Component Granularity

### Too Granular (Avoid)

```
components/
├── ButtonContainer.tsx
├── ButtonWrapper.tsx
├── ButtonContent.tsx
├── ButtonText.tsx
├── ButtonIcon.tsx
└── Button.tsx  # imports all of the above
```

### Right Balance

```
components/
└── Button.tsx  # handles all button variations via props
```

### Decision Framework

Break into separate components when:

- Logic is reused in multiple places
- Component exceeds ~150 lines
- A section has distinct responsibility (e.g., form vs. display)
- Testing requires isolation

Keep together when:

- Logic is only used once
- Breaking up adds cognitive overhead
- Components are tightly coupled anyway

---

## Folder Structure

### Feature-Based Organization

```
src/components/
├── ui/                    # Reusable UI primitives
│   ├── Button.tsx
│   ├── Card.tsx
│   ├── Input.tsx
│   ├── Toast.tsx
│   └── index.ts          # Re-exports
├── layout/               # Layout components
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── index.ts
├── auth/                 # Auth-related
│   ├── LoginButton.tsx
│   ├── UserMenu.tsx
│   └── index.ts
├── pages/                # Page-level components
│   ├── HomePage.tsx
│   ├── ProfilePage.tsx
│   └── index.ts
└── {feature}/            # Feature-specific
    ├── FeatureList.tsx
    ├── FeatureCard.tsx
    ├── FeatureForm.tsx
    └── index.ts
```

### Index Files for Clean Imports

```typescript
// src/components/ui/index.ts
export { Button } from './Button';
export { Card } from './Card';
export { Input } from './Input';
export { Toast } from './Toast';
```

```typescript
// Usage
import { Button, Card, Input } from '../components/ui';
```

---

## Loading and Skeleton Patterns

### Pattern: Progressive Rendering

Load at the smallest reasonable increment. Render static UI immediately, show skeletons for dynamic content.

```tsx
function KanbanBoard() {
  const { columns, loading } = useColumns();

  return (
    <div className="kanban-board">
      {/* Static: renders immediately */}
      <header>
        <h1>Project Board</h1>
        <AddColumnButton />
      </header>

      {/* Dynamic: skeleton → content */}
      <div className="columns">
        {loading ? (
          <ColumnsSkeleton count={3} />
        ) : (
          columns.map((col) => <Column key={col.id} column={col} />)
        )}
      </div>
    </div>
  );
}

function Column({ column }: { column: ColumnType }) {
  const { items, loading } = useColumnItems(column.id);

  return (
    <div className="column">
      {/* Column header renders immediately */}
      <h2>{column.name}</h2>

      {/* Items load per-column */}
      {loading ? (
        <ItemsSkeleton count={5} />
      ) : (
        items.map((item) => <Item key={item.id} item={item} />)
      )}
    </div>
  );
}
```

### Skeleton Components

```tsx
// src/components/ui/Skeleton.tsx
interface SkeletonProps {
  className?: string;
}

export function Skeleton({ className }: SkeletonProps) {
  return <div className={`animate-pulse bg-muted rounded ${className}`} />;
}

export function CardSkeleton() {
  return (
    <div className="p-4 border rounded-lg">
      <Skeleton className="h-4 w-3/4 mb-2" />
      <Skeleton className="h-3 w-full mb-1" />
      <Skeleton className="h-3 w-2/3" />
    </div>
  );
}

export function TableRowSkeleton() {
  return (
    <tr>
      <td>
        <Skeleton className="h-4 w-24" />
      </td>
      <td>
        <Skeleton className="h-4 w-32" />
      </td>
      <td>
        <Skeleton className="h-4 w-16" />
      </td>
    </tr>
  );
}
```

### Loading States in Lists

```tsx
function ItemsList() {
  const { items, loading, error } = useItems();

  if (error) {
    return <ErrorMessage message={error} />;
  }

  return (
    <div className="grid gap-4">
      {loading ? (
        Array.from({ length: 6 }).map((_, i) => <CardSkeleton key={i} />)
      ) : items.length === 0 ? (
        <EmptyState message="No items yet" />
      ) : (
        items.map((item) => <ItemCard key={item.id} item={item} />)
      )}
    </div>
  );
}
```

---

## Page-Level Components

### Pattern: Data Props from Parent

In Astro, the page fetches data and passes to React:

```astro
---
// src/pages/profile.astro
import { ProfilePage } from "../components/pages/ProfilePage";
import { getUserProfile } from "../lib/profile";

const profile = await getUserProfile(Astro.request);
---
<ProfilePage profile={profile} client:load />
```

```tsx
// src/components/pages/ProfilePage.tsx
interface ProfilePageProps {
  profile: Profile;
}

export function ProfilePage({ profile }: ProfilePageProps) {
  // Component receives data as props, doesn't fetch
  return (
    <div>
      <Avatar src={profile.image} />
      <h1>{profile.name}</h1>
      <ProfileForm profile={profile} />
    </div>
  );
}
```

### Pattern: Client-Side Data Fetching (Vite + React)

```tsx
// src/pages/ProfilePage.tsx
export function ProfilePage() {
  const { profile, loading, error, refetch } = useProfile();

  if (loading) return <ProfilePageSkeleton />;
  if (error) return <ErrorPage message={error} />;

  return (
    <div>
      <Avatar src={profile.image} />
      <h1>{profile.name}</h1>
      <ProfileForm profile={profile} onSuccess={refetch} />
    </div>
  );
}
```

---

## Modal and Dialog Patterns

### URL-Driven Modal

```tsx
function ItemsPage() {
  const [searchParams] = useSearchParams();
  const editId = searchParams.get('edit');
  const showCreate = searchParams.get('create') === 'true';

  return (
    <>
      <ItemsList />

      {showCreate && <CreateItemModal />}
      {editId && <EditItemModal itemId={editId} />}
    </>
  );
}
```

### State Over Previous Content

The modal overlays the list; the list maintains its scroll position:

```tsx
function EditItemModal({ itemId }: { itemId: string }) {
  const navigate = useNavigate();
  const { item, loading } = useItem(itemId);

  const handleClose = () => {
    navigate('/items', { replace: true });
  };

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center">
      <div className="bg-white p-6 rounded-lg max-w-md w-full">
        {loading ? <Skeleton className="h-48" /> : <ItemForm item={item} />}
        <Button onClick={handleClose}>Close</Button>
      </div>
    </div>
  );
}
```

---

## Component Props Patterns

### Composition Over Configuration

```tsx
// Good: Composition
<Card>
  <Card.Header>Title</Card.Header>
  <Card.Body>Content</Card.Body>
  <Card.Footer>
    <Button>Action</Button>
  </Card.Footer>
</Card>

// Also good: Simple props for simple cases
<Card title="Title" footer={<Button>Action</Button>}>
  Content
</Card>
```

### Polymorphic Components

```tsx
interface ButtonProps {
  as?: 'button' | 'a' | 'Link';
  href?: string;
  to?: string;
  // ...other props
}

function Button({ as = 'button', href, to, ...props }: ButtonProps) {
  if (as === 'a') return <a href={href} {...props} />;
  if (as === 'Link') return <Link to={to!} {...props} />;
  return <button {...props} />;
}
```

---

## Anti-Patterns

- **Over-abstraction** - Creating components for single-use cases
- **Prop drilling deep** - More than 2-3 levels suggests context or restructuring
- **God components** - 500+ line components that do everything
- **Fetching in deep children** - Fetch at page level, pass as props (Astro) or use hooks at appropriate level (React)
- **Inline styles everywhere** - Use Tailwind classes, extract repeated patterns

---

## Related Skills

- `astro-best-practices` - Astro component integration
- `vite-best-practices` - React patterns
- `feedback` - Toast and notification components
- `ui-design` - Visual design patterns
