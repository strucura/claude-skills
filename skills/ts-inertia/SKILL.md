---
name: ts-inertia
description: Create Inertia.js page components, layouts, shared data typing, and form handling for React. Use when the user wants to add or modify Inertia frontend pages.
argument-hint: "[page or layout name]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# Inertia Skill

You help create Inertia.js page components, layouts, shared data typing, and form handling for React frontends backed by Laravel. Inertia is the bridge — it eliminates the API layer by passing server-side data directly to React components as props.

## Core Principles

1. **Pages are full-page React components** — each page maps to a Laravel controller method. One page per route, no client-side routing.
2. **Props come from the server** — page props are defined by the controller (via Resources and `Inertia::render()`). The frontend types must match the backend response shape.
3. **Layouts wrap pages** — shared UI (nav, sidebar, footer) lives in layout components. Pages declare which layout they use.
4. **Forms use Inertia's `useForm`** — not raw fetch/axios. `useForm` handles submission, validation errors, processing state, and dirty tracking.
5. **Navigation uses `<Link>` or `router`** — Inertia intercepts navigation to avoid full page reloads.

## Discovery

Before creating Inertia pages:

1. **Find existing pages** — `Glob` for `**/pages/**/*.tsx` to discover the page directory structure and naming conventions.
2. **Find existing layouts** — `Grep` for `Layout` or `layout` in page files to discover layout patterns.
3. **Find the Inertia entry point** — `Glob` for `**/app.tsx` or `**/app.jsx` to understand the Inertia setup and how pages are resolved.
4. **Find the SSR entry point** — `Glob` for `**/ssr.tsx` to check if SSR is configured.
5. **Find shared types** — `Grep` for `PageProps` or `SharedProps` to discover how server-side data is typed.
6. **Find the corresponding controller** — read the backend controller to understand what props are passed and their shape.
7. **Find the corresponding Resource** — read the JsonResource to understand the exact data structure.

## Page Directory Structure

```
resources/js/
├── app.tsx                    # Inertia entry point
├── ssr.tsx                    # SSR entry point (if configured)
├── types/
│   ├── index.d.ts             # Shared type declarations
│   └── inertia.d.ts           # PageProps augmentation
├── layouts/
│   ├── AppLayout.tsx           # Authenticated layout
│   ├── GuestLayout.tsx         # Unauthenticated layout
│   └── SettingsLayout.tsx      # Nested layout for settings section
├── pages/
│   ├── assets/
│   │   ├── index.tsx           # Assets list page
│   │   ├── show.tsx            # Asset detail page
│   │   ├── create.tsx          # Asset creation form
│   │   └── edit.tsx            # Asset edit form
│   └── dashboard/
│       └── index.tsx           # Dashboard page
└── components/
    └── ui/                     # Shared UI components
```

### Naming Conventions

| Type | File name | Component name |
|---|---|---|
| Page | `resources/js/pages/assets/index.tsx` | `Index` (default export) |
| Layout | `resources/js/layouts/AppLayout.tsx` | `AppLayout` |
| Shared component | `resources/js/components/ui/Badge.tsx` | `Badge` |

Pages use **lowercase file names** matching the route segment. Layouts and components use **PascalCase**.

## Shared Types

### types/inertia.d.ts — PageProps Augmentation

```typescript
import type { PageProps as InertiaPageProps } from '@inertiajs/core';

declare module '@inertiajs/core' {
  interface PageProps extends InertiaPageProps {
    auth: {
      user: {
        id: number;
        name: string;
        email: string;
      };
    };
    flash: {
      success?: string;
      error?: string;
    };
  }
}
```

This augments `usePage().props` globally so `auth` and `flash` are always typed without per-page declarations.

### Page-Specific Props Interface

```typescript
import type { PaginatedResponse } from '@/types';

interface AssetIndexProps {
  assets: PaginatedResponse<{
    id: number;
    name: string;
    status: string;
    category: {
      id: number;
      name: string;
    } | null;
    created_at: string;
  }>;
  can: {
    create: boolean;
  };
}
```

**Rules for page props types:**
- Define the interface in the same file as the page component — not in a shared types file.
- The shape must match the Laravel Resource output exactly. Read the Resource to derive the type.
- `_at` fields are `string` (ISO 8601 from `toAtomString()`). Parse to `Date` in the component if needed.
- `_on` fields are `string` (YYYY-MM-DD from `format('Y-m-d')`).
- Relationships that use `whenLoaded()` are nullable — type them as `T | null`.

### Common Shared Types

```typescript
// types/index.d.ts

export interface PaginatedResponse<T> {
  data: T[];
  links: {
    first: string;
    last: string;
    prev: string | null;
    next: string | null;
  };
  meta: {
    current_page: number;
    from: number | null;
    last_page: number;
    per_page: number;
    to: number | null;
    total: number;
    links: Array<{
      url: string | null;
      label: string;
      active: boolean;
    }>;
  };
}

export interface SelectOption {
  label: string;
  value: string | number;
}
```

## Page Component Templates

### Index Page (List)

```typescript
import { Head, Link } from '@inertiajs/react';
import type { PaginatedResponse } from '@/types';
import AppLayout from '@/layouts/AppLayout';

interface AssetIndexProps {
  assets: PaginatedResponse<{
    id: number;
    name: string;
    status: string;
    created_at: string;
  }>;
  can: {
    create: boolean;
  };
}

export default function Index({ assets, can }: AssetIndexProps) {
  return (
    <AppLayout>
      <Head title="Assets" />

      <div>
        <h1>Assets</h1>

        {can.create && (
          <Link href={route('assets.create')}>Create Asset</Link>
        )}

        <ul>
          {assets.data.map((asset) => (
            <li key={asset.id}>
              <Link href={route('assets.show', asset.id)}>
                {asset.name}
              </Link>
            </li>
          ))}
        </ul>
      </div>
    </AppLayout>
  );
}
```

### Show Page (Detail)

Same structure as Index. Props contain a single object instead of `PaginatedResponse`. Use `can` for conditional edit/delete links.

```typescript
interface AssetShowProps {
  asset: {
    id: number;
    name: string;
    description: string | null;
    status: string;
    category: { id: number; name: string } | null;
    created_at: string;
  };
  can: {
    edit: boolean;
    destroy: boolean;
  };
}
```

### Create/Edit Page (Form)

```typescript
import { Head, useForm, router } from '@inertiajs/react';
import type { FormEventHandler } from 'react';
import AppLayout from '@/layouts/AppLayout';
import type { SelectOption } from '@/types';

interface AssetCreateProps {
  categories: SelectOption[];
}

export default function Create({ categories }: AssetCreateProps) {
  const { data, setData, post, processing, errors, reset } = useForm({
    name: '',
    description: '',
    status: 'draft',
    category_id: '',
  });

  const submit: FormEventHandler = (e) => {
    e.preventDefault();
    post(route('assets.store'), {
      onSuccess: () => reset(),
    });
  };

  return (
    <AppLayout>
      <Head title="Create Asset" />

      <form onSubmit={submit}>
        <div>
          <label htmlFor="name">Name</label>
          <input
            id="name"
            type="text"
            value={data.name}
            onChange={(e) => setData('name', e.target.value)}
          />
          {errors.name && <p>{errors.name}</p>}
        </div>

        {/* Repeat pattern for additional fields */}

        <div>
          <label htmlFor="category_id">Category</label>
          <select
            id="category_id"
            value={data.category_id}
            onChange={(e) => setData('category_id', e.target.value)}
          >
            <option value="">Select a category</option>
            {categories.map((cat) => (
              <option key={cat.value} value={cat.value}>
                {cat.label}
              </option>
            ))}
          </select>
          {errors.category_id && <p>{errors.category_id}</p>}
        </div>

        <button type="submit" disabled={processing}>
          Create Asset
        </button>
      </form>
    </AppLayout>
  );
}
```

### Edit Page (Pre-filled Form)

Same structure as Create, with two differences:
1. **Props include the existing model** — `useForm` initializes from `asset` prop values (use `?? ''` for nullable fields)
2. **Uses `put` instead of `post`** — `put(route('assets.update', asset.id))`

## Layout Components

### Persistent Layout

```typescript
import { Link, usePage } from '@inertiajs/react';
import type { ReactNode } from 'react';

interface AppLayoutProps {
  children: ReactNode;
}

export default function AppLayout({ children }: AppLayoutProps) {
  const { auth, flash } = usePage().props;

  return (
    <div>
      <nav>
        <Link href={route('dashboard')}>Dashboard</Link>
        <Link href={route('assets.index')}>Assets</Link>
        <span>{auth.user.name}</span>
      </nav>

      {flash.success && (
        <div role="alert">{flash.success}</div>
      )}

      {flash.error && (
        <div role="alert">{flash.error}</div>
      )}

      <main>{children}</main>
    </div>
  );
}
```

For sections with sub-navigation (e.g., Settings), create a nested layout that wraps `AppLayout` and adds sidebar nav.

## Inertia Form Handling

### useForm API

```typescript
const {
  data, setData, post, put, patch, delete: destroy,
  processing, errors, hasErrors, reset, clearErrors,
  isDirty, transform,
} = useForm({ name: '', email: '' });
```

### Form Submission Options

`post`/`put`/`patch`/`delete` accept an options object:
- **`onSuccess`** — runs after successful submission (e.g., `reset()`)
- **`onError`** — runs when validation errors are returned
- **`onFinish`** — runs after completion regardless of outcome
- **`preserveScroll: true`** — don't scroll to top on success
- **`preserveState: true`** — keep component state on validation error (default for POST/PUT/PATCH)

Use `transform((data) => ({ ...data, field: transformed }))` before submission to modify data without changing form state.

For delete operations, use `router.delete(route('assets.destroy', id))` directly (no `useForm` needed).

## Navigation

### Link Component

```typescript
import { Link } from '@inertiajs/react';

// Basic link
<Link href={route('assets.index')}>Assets</Link>

// With method (for non-GET requests embedded in the page)
<Link href={route('assets.destroy', asset.id)} method="delete" as="button">
  Delete
</Link>

// Preserve scroll position
<Link href={route('assets.index', { page: 2 })} preserveScroll>
  Next Page
</Link>

// Active state detection
<Link
  href={route('assets.index')}
  className={route().current('assets.*') ? 'active' : ''}
>
  Assets
</Link>
```

### Programmatic Navigation

Use `router.visit()` for navigation, `router.reload()` to re-fetch props (with optional `{ only: ['assets'] }` for partial reloads), and `{ replace: true }` to replace the history entry.

## Ziggy Route Helper

Routes are generated by Ziggy (`ziggy-js`) and available globally via the `route()` function:

```typescript
route('assets.index')                    // /assets
route('assets.show', asset.id)           // /assets/1
route().current('assets.*')              // true if current route matches
```

## What This Skill Does NOT Cover

- **Backend controllers, routes, or resources** → use `controller`, `form-request`, `resource` skills
- **React component testing** → use `react-test` skill

## Checklist

When creating an Inertia page:

1. **Read the corresponding controller** — understand what props are passed via `Inertia::render()`
2. **Read the corresponding Resource** — derive the TypeScript type from the Resource's `toArray()` output
3. **Define the props interface** — in the same file as the page component, matching the Resource shape exactly
4. **Choose the layout** — `AppLayout` for authenticated pages, `GuestLayout` for public, nested layouts for sections
5. **Use `useForm` for forms** — never raw fetch. Handle `errors`, `processing`, and `reset`
6. **Use `<Link>` for navigation** — never `<a href>` for internal routes
7. **Use `route()` for URLs** — never hardcode paths
8. **Handle the `can` object** — conditionally show/hide UI based on permissions passed from the controller
9. **Handle flash messages** — display `flash.success` and `flash.error` from the layout
