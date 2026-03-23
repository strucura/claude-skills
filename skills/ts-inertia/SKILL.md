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
5. **Navigation uses `<Link>` or `router`** — never `<a href>` for internal links. Inertia intercepts navigation to avoid full page reloads.

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

```typescript
import { Head, Link } from '@inertiajs/react';
import AppLayout from '@/layouts/AppLayout';

interface AssetShowProps {
  asset: {
    id: number;
    name: string;
    description: string | null;
    status: string;
    category: {
      id: number;
      name: string;
    } | null;
    created_at: string;
  };
  can: {
    edit: boolean;
    destroy: boolean;
  };
}

export default function Show({ asset, can }: AssetShowProps) {
  return (
    <AppLayout>
      <Head title={asset.name} />

      <div>
        <h1>{asset.name}</h1>
        {asset.description && <p>{asset.description}</p>}
        <p>Status: {asset.status}</p>
        {asset.category && <p>Category: {asset.category.name}</p>}

        {can.edit && (
          <Link href={route('assets.edit', asset.id)}>Edit</Link>
        )}
      </div>
    </AppLayout>
  );
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

        <div>
          <label htmlFor="description">Description</label>
          <textarea
            id="description"
            value={data.description}
            onChange={(e) => setData('description', e.target.value)}
          />
          {errors.description && <p>{errors.description}</p>}
        </div>

        <div>
          <label htmlFor="status">Status</label>
          <select
            id="status"
            value={data.status}
            onChange={(e) => setData('status', e.target.value)}
          >
            <option value="draft">Draft</option>
            <option value="active">Active</option>
          </select>
          {errors.status && <p>{errors.status}</p>}
        </div>

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

```typescript
import { Head, useForm } from '@inertiajs/react';
import type { FormEventHandler } from 'react';
import AppLayout from '@/layouts/AppLayout';
import type { SelectOption } from '@/types';

interface AssetEditProps {
  asset: {
    id: number;
    name: string;
    description: string | null;
    status: string;
    category_id: number | null;
  };
  categories: SelectOption[];
}

export default function Edit({ asset, categories }: AssetEditProps) {
  const { data, setData, put, processing, errors } = useForm({
    name: asset.name,
    description: asset.description ?? '',
    status: asset.status,
    category_id: asset.category_id?.toString() ?? '',
  });

  const submit: FormEventHandler = (e) => {
    e.preventDefault();
    put(route('assets.update', asset.id));
  };

  return (
    <AppLayout>
      <Head title={`Edit ${asset.name}`} />

      <form onSubmit={submit}>
        {/* Same fields as create, pre-filled from asset */}
        <button type="submit" disabled={processing}>
          Update Asset
        </button>
      </form>
    </AppLayout>
  );
}
```

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

### Nested Layout

For sections with sub-navigation (e.g., Settings):

```typescript
import { Link } from '@inertiajs/react';
import type { ReactNode } from 'react';
import AppLayout from './AppLayout';

interface SettingsLayoutProps {
  children: ReactNode;
}

export default function SettingsLayout({ children }: SettingsLayoutProps) {
  return (
    <AppLayout>
      <div>
        <aside>
          <nav>
            <Link href={route('settings.profile')}>Profile</Link>
            <Link href={route('settings.security')}>Security</Link>
            <Link href={route('settings.team')}>Team</Link>
          </nav>
        </aside>

        <div>{children}</div>
      </div>
    </AppLayout>
  );
}
```

## Inertia Form Handling

### useForm API

```typescript
const {
  data,           // Current form data
  setData,        // Update a field: setData('name', value) or setData({ ...overrides })
  post,           // Submit as POST
  put,            // Submit as PUT
  patch,          // Submit as PATCH
  delete: destroy,// Submit as DELETE (renamed to avoid reserved word)
  processing,     // Boolean — true while submitting
  errors,         // Server validation errors keyed by field name
  hasErrors,      // Boolean — true if any errors
  reset,          // Reset to initial values: reset() or reset('name', 'email')
  clearErrors,    // Clear errors: clearErrors() or clearErrors('name')
  isDirty,        // Boolean — true if data differs from initial values
  transform,      // Transform data before submission
} = useForm({
  name: '',
  email: '',
});
```

### Form Submission Options

```typescript
post(route('assets.store'), {
  onSuccess: () => {
    // Runs after successful submission (no validation errors)
    reset();
  },
  onError: (errors) => {
    // Runs when validation errors are returned
    // errors is the same as the errors object from useForm
  },
  onFinish: () => {
    // Runs after submission completes (success or error)
  },
  preserveScroll: true,    // Don't scroll to top on success
  preserveState: true,     // Keep component state on validation error (default for POST/PUT/PATCH)
});
```

### Data Transformation

```typescript
const { data, setData, post, transform } = useForm({
  name: '',
  tags: [] as string[],
});

const submit: FormEventHandler = (e) => {
  e.preventDefault();

  transform((data) => ({
    ...data,
    tags: data.tags.filter(Boolean), // Remove empty strings
  }));

  post(route('assets.store'));
};
```

### Delete with Confirmation

```typescript
import { router } from '@inertiajs/react';

function handleDelete(assetId: number) {
  if (confirm('Are you sure?')) {
    router.delete(route('assets.destroy', assetId));
  }
}
```

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

```typescript
import { router } from '@inertiajs/react';

// Visit a page
router.visit(route('assets.show', asset.id));

// Reload the current page (re-fetch props)
router.reload();

// Reload only specific props
router.reload({ only: ['assets'] });

// Replace history entry (back button skips this page)
router.visit(route('assets.index'), { replace: true });
```

## Ziggy Route Helper

Routes are generated by Ziggy (`ziggy-js`) and available globally via the `route()` function:

```typescript
route('assets.index')                    // /assets
route('assets.show', 1)                  // /assets/1
route('assets.show', { asset: 1 })       // /assets/1
route('assets.index', { page: 2 })       // /assets?page=2
route().current('assets.*')              // true if current route matches
route().current('assets.show', { asset: 1 }) // true if on this exact route
```

## What This Skill Does NOT Cover

- **Backend controllers, routes, or resources** → use `controller`, `form-request`, `resource` skills
- **React component testing** → use `react-test` skill
- **Package creation** → use `ts-package` skill
- **Standalone React components not tied to Inertia** → write standard React components

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
