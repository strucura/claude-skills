---
name: shadcn-component
description: Create and customize shadcn/ui-style components with CVA variants, Radix primitives, and Tailwind styling. Use when the user wants to add or modify UI components.
argument-hint: "[component name or description]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, WebSearch
---

# shadcn Component Skill

You help create UI components following shadcn/ui conventions — Radix primitives for accessibility, CVA for type-safe variants, Tailwind CSS for styling, and `cn()` for class merging. Components are unstyled at the primitive level and fully customizable by the consumer.

## Core Principles

1. **Accessible by default** — every component is built on Radix UI primitives or native HTML elements with correct ARIA roles, keyboard navigation, and focus management.
2. **Variants via CVA** — visual variations (variant, size, color) are defined with `class-variance-authority`. No conditional class string building.
3. **`cn()` for class merging** — always use `cn()` (clsx + tailwind-merge) so consumer `className` overrides resolve Tailwind conflicts correctly.
4. **Props extend native elements** — components accept all native HTML or Radix props via `React.ComponentProps<>`. Never restrict what the consumer can pass.
5. **Copy-paste ownership** — components live in the consumer's codebase (`components/ui/`). They are meant to be read, understood, and modified.

## Discovery

Before creating a component, scan the current project to understand its conventions:

1. **Find components.json** — `Glob` for `**/components.json` to discover the shadcn configuration (style, aliases, icon library, Tailwind CSS config).
2. **Find the `cn()` utility** — `Grep` for `export function cn` to locate `lib/utils.ts`. If it doesn't exist, create it.
3. **Find existing UI components** — `Glob` for `**/components/ui/*.tsx` to discover the directory structure, naming patterns, and conventions in use.
4. **Find Radix imports** — `Grep` for `@radix-ui` to see which primitives are already installed.
5. **Find CVA usage** — `Grep` for `import.*cva` to confirm class-variance-authority is in use and see how variants are structured.
6. **Check Tailwind version** — read `package.json` to determine if the project uses Tailwind v3 (config file) or v4 (CSS-based config).

## The `cn()` Utility

Every project using shadcn components needs this utility. If it doesn't exist, create it:

### lib/utils.ts

```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**Required packages:** `clsx`, `tailwind-merge`

## Component Templates

### Simple Component (No Radix)

For components that wrap a native HTML element with styling and variants:

```typescript
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const badgeVariants = cva(
  'inline-flex items-center rounded-md border px-2.5 py-0.5 text-xs font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2',
  {
    variants: {
      variant: {
        default: 'border-transparent bg-primary text-primary-foreground shadow',
        secondary: 'border-transparent bg-secondary text-secondary-foreground',
        destructive: 'border-transparent bg-destructive text-destructive-foreground shadow',
        outline: 'text-foreground',
      },
    },
    defaultVariants: {
      variant: 'default',
    },
  },
);

function Badge({
  className,
  variant,
  ...props
}: React.ComponentProps<'div'> & VariantProps<typeof badgeVariants>) {
  return (
    <div
      data-slot="badge"
      className={cn(badgeVariants({ variant }), className)}
      {...props}
    />
  );
}

export { Badge, badgeVariants };
```

### Component with CVA Variants and Sizes

```typescript
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground shadow hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground shadow-sm hover:bg-destructive/90',
        // Add more variants as needed
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 rounded-md px-3 text-xs',
        // Add more sizes as needed
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  },
);

function Button({
  className,
  variant,
  size,
  asChild = false,
  ...props
}: React.ComponentProps<'button'> &
  VariantProps<typeof buttonVariants> & {
    asChild?: boolean;
  }) {
  const Comp = asChild ? Slot : 'button';

  return (
    <Comp
      data-slot="button"
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  );
}

export { Button, buttonVariants };
```

### Radix Primitive Wrapper

For components built on a Radix UI primitive:

```typescript
import * as CheckboxPrimitive from '@radix-ui/react-checkbox';
import { CheckIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

function Checkbox({
  className,
  ...props
}: React.ComponentProps<typeof CheckboxPrimitive.Root>) {
  return (
    <CheckboxPrimitive.Root
      data-slot="checkbox"
      className={cn(
        'peer h-4 w-4 shrink-0 rounded-sm border border-primary shadow focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 data-[state=checked]:bg-primary data-[state=checked]:text-primary-foreground',
        className,
      )}
      {...props}
    >
      <CheckboxPrimitive.Indicator className={cn('flex items-center justify-center text-current')}>
        <CheckIcon className="h-4 w-4" />
      </CheckboxPrimitive.Indicator>
    </CheckboxPrimitive.Root>
  );
}

export { Checkbox };
```

**Radix import convention:** Always import the namespace (`import * as CheckboxPrimitive`) not individual parts. This makes it clear which parts come from Radix vs custom code.

### Compound Component

For multi-part components where each piece is exported separately:

```typescript
import { cn } from '@/lib/utils';

function Card({ className, ...props }: React.ComponentProps<'div'>) {
  return (
    <div
      data-slot="card"
      className={cn('rounded-xl border bg-card text-card-foreground shadow', className)}
      {...props}
    />
  );
}

function CardHeader({ className, ...props }: React.ComponentProps<'div'>) {
  return (
    <div
      data-slot="card-header"
      className={cn('flex flex-col space-y-1.5 p-6', className)}
      {...props}
    />
  );
}

// Repeat for CardTitle, CardDescription, CardContent, CardFooter — same pattern with different data-slot names and base classes.

export { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter };
```

### Complex Radix Compound Component

For multi-part Radix primitives (Select, DropdownMenu, Dialog), apply the same pattern as the simpler examples above, but for each sub-component:

1. Import `* as XPrimitive` from the Radix package
2. Wrap each Radix part (Root, Trigger, Content, Item, etc.) in its own function
3. Add `data-slot`, `cn()` for className merging, and spread `{...props}`
4. Use `SelectPrimitive.Portal` for content that renders in a portal
5. Style state with Radix data attributes: `data-[state=open]:animate-in`, `data-[disabled]:opacity-50`

Export all parts as named exports.

### Component with forwardRef

Use `forwardRef` when the consumer needs ref access to the underlying DOM node — typically for form integration, focus management, or imperative APIs:

```typescript
import * as React from 'react';
import { cn } from '@/lib/utils';

const Input = React.forwardRef<HTMLInputElement, React.ComponentProps<'input'>>(
  ({ className, type, ...props }, ref) => {
    return (
      <input
        type={type}
        data-slot="input"
        className={cn(
          'flex h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-sm transition-colors file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50',
          className,
        )}
        ref={ref}
        {...props}
      />
    );
  },
);

Input.displayName = 'Input';

export { Input };
```

Use forwardRef for form inputs and components needing imperative ref access. Skip for simple display components (Badge, Card).

### Component with Internal Context

For components needing shared state across compound parts, follow the context + provider pattern from the `ts-package` skill.

## CVA Patterns

CVA structure: `cva('base-classes', { variants: { ... }, compoundVariants: [...], defaultVariants: { ... } })`

- **`VariantProps<typeof xVariants>`** extracts variant types. Intersect with `React.ComponentProps<>` for the full props type.
- **`compoundVariants`** apply classes when multiple variant values match simultaneously.
- **Always export variants alongside the component** (`export { Button, buttonVariants }`) so consumers can generate classes without rendering (e.g., on `<Link>` elements).

## The `asChild` / Slot Pattern

The `asChild` prop replaces the component's root element with the child element, merging props and refs. Powered by `@radix-ui/react-slot`.

```typescript
import { Slot } from '@radix-ui/react-slot';

function Button({ asChild = false, ...props }) {
  const Comp = asChild ? Slot : 'button';
  return <Comp {...props} />;
}

// Renders as <button>
<Button>Click me</Button>

// Renders as <a> with all Button props merged onto it
<Button asChild>
  <a href="/about">Click me</a>
</Button>
```

Support `asChild` on interactive components that may render as a different element. Skip for display-only components.

## `data-slot` Convention

Every component part gets `data-slot` (kebab-case, matching component name). Used for CSS targeting, DevTools debugging, and test queries. Prefer role queries first.

## Radix Data Attributes

Radix primitives expose state via `data-*` attributes. Use these for styling instead of managing state classes manually:

```typescript
// Radix automatically adds data-state="checked" / data-state="unchecked"
className="data-[state=checked]:bg-primary data-[state=checked]:text-primary-foreground"

// Radix adds data-state="open" / data-state="closed"
className="data-[state=open]:animate-in data-[state=closed]:animate-out"

// Radix adds data-disabled when disabled
className="data-[disabled]:pointer-events-none data-[disabled]:opacity-50"

// Radix adds data-side and data-align for positioned content
className="data-[side=bottom]:translate-y-1 data-[side=top]:-translate-y-1"
```

## CSS Variable Tokens

shadcn components use CSS variable tokens from the project's Tailwind theme. These are the semantic color tokens:

| Token | Purpose |
|---|---|
| `bg-background` / `text-foreground` | Page background and default text |
| `bg-card` / `text-card-foreground` | Card surfaces |
| `bg-popover` / `text-popover-foreground` | Popovers, dropdowns, tooltips |
| `bg-primary` / `text-primary-foreground` | Primary actions (buttons, links) |
| `bg-secondary` / `text-secondary-foreground` | Secondary actions |
| `bg-muted` / `text-muted-foreground` | Subdued content, disabled states |
| `bg-accent` / `text-accent-foreground` | Hover/active states |
| `bg-destructive` / `text-destructive-foreground` | Destructive actions, errors |
| `border-input` | Form input borders |
| `ring-ring` | Focus ring color |

Always use these semantic tokens — never hardcode colors. This ensures the component works across themes.

## Icon Integration

Use `lucide-react` for icons (unless the project's `components.json` specifies a different `iconLibrary`):

```typescript
import { CheckIcon, ChevronDownIcon, XIcon } from 'lucide-react';

// Inside a component
<CheckIcon className="h-4 w-4" />

// As a Radix icon slot
<SelectPrimitive.Icon asChild>
  <ChevronDownIcon className="h-4 w-4 opacity-50" />
</SelectPrimitive.Icon>
```

**Sizing convention:** `h-4 w-4` for inline/small, `h-5 w-5` for medium, `h-6 w-6` for large.

## File Structure and Naming

| Type | Location | File name | Example |
|---|---|---|---|
| UI component | `components/ui/` | `{component}.tsx` (kebab-case) | `components/ui/button.tsx` |
| Utility | `lib/` | `utils.ts` | `lib/utils.ts` |

**One file per component** for simple components (Button, Badge, Input).

**Directory per component** for complex components with multiple files:

```
components/ui/
  select.tsx           # Simple — one file
  button.tsx           # Simple — one file
  data-grid/           # Complex — directory
    data-grid.tsx
    data-grid-header.tsx
    data-grid-row.tsx
    index.ts           # Barrel export
```

## Checklist

When creating a shadcn component:

1. **Check if `cn()` exists** — create `lib/utils.ts` if it doesn't
2. **Identify the Radix primitive** — if the component needs accessibility behavior (checkbox, select, dialog, etc.), wrap the Radix primitive. If it's purely visual (badge, card), use native HTML.
3. **Define CVA variants** — if the component has visual variations. Export the variants alongside the component.
4. **Support `asChild`** — if the component is interactive and might render as a different element.
5. **Add `data-slot`** — on every component part.
6. **Use semantic color tokens** — `bg-primary`, `text-muted-foreground`, etc.
7. **Accept `className`** — destructure it and pass through `cn()` for proper merging.
8. **Spread remaining props** — `{...props}` on the root element.
9. **Use `forwardRef`** — for form inputs and components that need imperative ref access.
10. **Add `displayName`** — on any forwardRef component.
11. **Export as named exports** — component and variants.
