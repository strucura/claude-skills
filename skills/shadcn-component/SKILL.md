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

**Why both clsx and twMerge:**
- `clsx` handles conditional classes: `clsx('base', false && 'hidden', className)`
- `twMerge` resolves Tailwind conflicts: `twMerge('px-4', 'px-6')` → `'px-6'` (not `'px-4 px-6'`)
- Together they ensure consumer `className` overrides work correctly.

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
        outline: 'border border-input bg-background shadow-sm hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground shadow-sm hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 rounded-md px-3 text-xs',
        lg: 'h-10 rounded-md px-8',
        icon: 'h-9 w-9',
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

function CardTitle({ className, ...props }: React.ComponentProps<'div'>) {
  return (
    <div
      data-slot="card-title"
      className={cn('font-semibold leading-none tracking-tight', className)}
      {...props}
    />
  );
}

function CardDescription({ className, ...props }: React.ComponentProps<'div'>) {
  return (
    <div
      data-slot="card-description"
      className={cn('text-sm text-muted-foreground', className)}
      {...props}
    />
  );
}

function CardContent({ className, ...props }: React.ComponentProps<'div'>) {
  return (
    <div
      data-slot="card-content"
      className={cn('p-6 pt-0', className)}
      {...props}
    />
  );
}

function CardFooter({ className, ...props }: React.ComponentProps<'div'>) {
  return (
    <div
      data-slot="card-footer"
      className={cn('flex items-center p-6 pt-0', className)}
      {...props}
    />
  );
}

export { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter };
```

### Complex Radix Compound Component

For multi-part Radix primitives like Select, DropdownMenu, Dialog:

```typescript
import * as SelectPrimitive from '@radix-ui/react-select';
import { CheckIcon, ChevronDownIcon, ChevronUpIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

function Select({ ...props }: React.ComponentProps<typeof SelectPrimitive.Root>) {
  return <SelectPrimitive.Root data-slot="select" {...props} />;
}

function SelectTrigger({
  className,
  children,
  ...props
}: React.ComponentProps<typeof SelectPrimitive.Trigger>) {
  return (
    <SelectPrimitive.Trigger
      data-slot="select-trigger"
      className={cn(
        'flex h-9 w-full items-center justify-between rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm ring-offset-background placeholder:text-muted-foreground focus:outline-none focus:ring-1 focus:ring-ring disabled:cursor-not-allowed disabled:opacity-50',
        className,
      )}
      {...props}
    >
      {children}
      <SelectPrimitive.Icon asChild>
        <ChevronDownIcon className="h-4 w-4 opacity-50" />
      </SelectPrimitive.Icon>
    </SelectPrimitive.Trigger>
  );
}

function SelectContent({
  className,
  children,
  position = 'popper',
  ...props
}: React.ComponentProps<typeof SelectPrimitive.Content>) {
  return (
    <SelectPrimitive.Portal>
      <SelectPrimitive.Content
        data-slot="select-content"
        className={cn(
          'relative z-50 max-h-96 min-w-[8rem] overflow-hidden rounded-md border bg-popover text-popover-foreground shadow-md data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95',
          position === 'popper' &&
            'data-[side=bottom]:translate-y-1 data-[side=left]:-translate-x-1 data-[side=right]:translate-x-1 data-[side=top]:-translate-y-1',
          className,
        )}
        position={position}
        {...props}
      >
        <SelectPrimitive.Viewport
          className={cn(
            'p-1',
            position === 'popper' &&
              'h-[var(--radix-select-trigger-height)] w-full min-w-[var(--radix-select-trigger-width)]',
          )}
        >
          {children}
        </SelectPrimitive.Viewport>
      </SelectPrimitive.Content>
    </SelectPrimitive.Portal>
  );
}

function SelectItem({
  className,
  children,
  ...props
}: React.ComponentProps<typeof SelectPrimitive.Item>) {
  return (
    <SelectPrimitive.Item
      data-slot="select-item"
      className={cn(
        'relative flex w-full cursor-default select-none items-center rounded-sm py-1.5 pl-2 pr-8 text-sm outline-none focus:bg-accent focus:text-accent-foreground data-[disabled]:pointer-events-none data-[disabled]:opacity-50',
        className,
      )}
      {...props}
    >
      <span className="absolute right-2 flex h-3.5 w-3.5 items-center justify-center">
        <SelectPrimitive.ItemIndicator>
          <CheckIcon className="h-4 w-4" />
        </SelectPrimitive.ItemIndicator>
      </span>
      <SelectPrimitive.ItemText>{children}</SelectPrimitive.ItemText>
    </SelectPrimitive.Item>
  );
}

function SelectValue({ ...props }: React.ComponentProps<typeof SelectPrimitive.Value>) {
  return <SelectPrimitive.Value data-slot="select-value" {...props} />;
}

export { Select, SelectTrigger, SelectContent, SelectItem, SelectValue };
```

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

**When to use forwardRef:**
- Form inputs (Input, Textarea, Select triggers) — React Hook Form and other form libraries need refs.
- Components that manage focus programmatically.
- Components that wrap third-party libraries requiring a ref.

**When NOT to use forwardRef:**
- Simple display components (Badge, Card) — no ref needed.
- Compound component wrappers (Card, CardHeader) — consumers use refs on the inner parts, not the wrapper.

### Component with Internal Context

For complex components that need shared state across compound parts:

```typescript
import * as React from 'react';
import { cn } from '@/lib/utils';

type SidebarState = 'expanded' | 'collapsed';

interface SidebarContextValue {
  state: SidebarState;
  open: boolean;
  setOpen: (open: boolean) => void;
  toggleSidebar: () => void;
}

const SidebarContext = React.createContext<SidebarContextValue | null>(null);

function useSidebar(): SidebarContextValue {
  const context = React.useContext(SidebarContext);
  if (!context) {
    throw new Error('useSidebar must be used within a <SidebarProvider>');
  }
  return context;
}

interface SidebarProviderProps extends React.ComponentProps<'div'> {
  defaultOpen?: boolean;
}

function SidebarProvider({
  defaultOpen = true,
  className,
  children,
  ...props
}: SidebarProviderProps) {
  const [open, setOpen] = React.useState(defaultOpen);

  const value = React.useMemo<SidebarContextValue>(
    () => ({
      state: open ? 'expanded' : 'collapsed',
      open,
      setOpen,
      toggleSidebar: () => setOpen((prev) => !prev),
    }),
    [open],
  );

  return (
    <SidebarContext.Provider value={value}>
      <div data-slot="sidebar-provider" className={cn('flex min-h-screen', className)} {...props}>
        {children}
      </div>
    </SidebarContext.Provider>
  );
}

function Sidebar({ className, ...props }: React.ComponentProps<'aside'>) {
  const { state } = useSidebar();

  return (
    <aside
      data-slot="sidebar"
      data-state={state}
      className={cn(
        'flex w-64 flex-col border-r bg-sidebar text-sidebar-foreground transition-all data-[state=collapsed]:w-16',
        className,
      )}
      {...props}
    />
  );
}

function SidebarTrigger({ className, ...props }: React.ComponentProps<'button'>) {
  const { toggleSidebar } = useSidebar();

  return (
    <button
      data-slot="sidebar-trigger"
      onClick={toggleSidebar}
      className={cn('inline-flex h-9 w-9 items-center justify-center rounded-md', className)}
      {...props}
    />
  );
}

export { SidebarProvider, Sidebar, SidebarTrigger, useSidebar };
```

## CVA Patterns

### Defining Variants

```typescript
const variants = cva(
  'base-classes-that-always-apply',
  {
    variants: {
      variant: {
        default: 'classes-for-default',
        destructive: 'classes-for-destructive',
      },
      size: {
        default: 'h-9 px-4',
        sm: 'h-8 px-3 text-xs',
        lg: 'h-10 px-8',
      },
    },
    compoundVariants: [
      {
        variant: 'destructive',
        size: 'lg',
        className: 'font-bold',
      },
    ],
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  },
);
```

### Typing with VariantProps

```typescript
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva('...', { variants: { ... } });

type ButtonProps = React.ComponentProps<'button'> &
  VariantProps<typeof buttonVariants> & {
    asChild?: boolean;
  };
```

`VariantProps` extracts the variant types from a CVA definition. Combined with `React.ComponentProps`, the component accepts all native props plus the typed variant options.

### Exporting Variants

Always export the CVA variants alongside the component so consumers can use them programmatically:

```typescript
export { Button, buttonVariants };
```

This allows consumers to generate classes without rendering:

```typescript
import { buttonVariants } from '@/components/ui/button';

<Link className={buttonVariants({ variant: 'outline' })} href="/about">
  About
</Link>
```

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

**When to support `asChild`:**
- Interactive components that might need to render as a different element (Button → Link, Button → anchor).
- Trigger components in Radix compound patterns.

**When NOT to support `asChild`:**
- Display-only components (Badge, Card, CardContent).
- Components where changing the root element doesn't make sense.

## `data-slot` Convention

Every component part gets a `data-slot` attribute matching its semantic role:

```typescript
<div data-slot="card" />
<div data-slot="card-header" />
<div data-slot="card-title" />
<div data-slot="card-content" />
<div data-slot="card-footer" />
```

**Purpose:**
- CSS targeting: `[data-slot="card-header"] { ... }` for parent-aware styling.
- Debugging: instantly identify component parts in DevTools.
- Testing: fallback query target when no accessible role exists (prefer role queries first).

**Naming:** kebab-case, matching the component name in lowercase.

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

## Component Conventions Summary

1. **Props type** — `React.ComponentProps<'element'>` or `React.ComponentProps<typeof RadixPrimitive.Part>`. Intersect with `VariantProps<>` and custom props.
2. **Destructure `className` first** — always extract it before the spread so `cn()` can merge it.
3. **Spread remaining props last** — `{...props}` on the root element passes through all native attributes.
4. **`data-slot` on every part** — kebab-case name matching the component.
5. **Named exports only** — `export { Button, buttonVariants }`. No default exports.
6. **`displayName` on forwardRef** — required for DevTools and error messages.
7. **Semantic color tokens** — use `bg-primary`, `text-muted-foreground`, etc. Never hardcode colors.
8. **Radix data attributes for state** — `data-[state=open]:...` instead of conditional classes.

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
