---
name: react-test
description: Create React component and hook tests with Testing Library. Use when the user wants to test React components, custom hooks, context providers, or user interactions.
argument-hint: "[component or hook to test]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# React Test Skill

You help create tests for React components, custom hooks, and context providers using `@testing-library/react`. This skill builds on the `ts-test` skill's Vitest/Jest patterns and adds the React-specific rendering, interaction, and assertion layer.

## Core Principles

1. **Test what the user sees and does** — query by role, text, and label. Never query by CSS class, tag name, or test ID unless there is no accessible alternative.
2. **Avoid testing implementation details** — don't assert on state values, component instances, or internal method calls. If the user can't observe it, don't test it.
3. **Every test renders a component or hook** — if it doesn't call `render()` or `renderHook()`, it belongs in the `ts-test` skill, not here.
4. **User events over fire events** — use `@testing-library/user-event` for interactions. `fireEvent` bypasses browser behavior (focus, blur, input events). `userEvent` simulates real user actions.
5. **Async by default** — user events and state updates are asynchronous. Always `await` user event calls and use `waitFor` / `findBy*` for assertions that depend on state changes.

## Discovery

Before writing React tests:

1. **Find the test setup file** — `Glob` for `**/test-setup.ts` or `**/setupTests.ts` to see what's globally configured.
2. **Find existing React tests** — `Glob` for `**/*.test.tsx` to discover patterns and conventions.
3. **Find test utilities** — `Grep` for `renderWithProviders` or `createWrapper` to discover custom render helpers.
4. **Read the component/hook** — understand its props, state, context dependencies, and user interactions.

## Test Setup

### src/test-setup.ts

```typescript
import '@testing-library/jest-dom/vitest';
```

This registers DOM matchers (`toBeInTheDocument`, `toBeVisible`, `toHaveTextContent`, etc.) for Vitest. For Jest, use:

```typescript
import '@testing-library/jest-dom';
```

## Component Testing

### Basic Component Test

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { Badge } from './Badge';

describe('Badge', () => {
  it('renders the label text', () => {
    render(<Badge label="Active" />);

    expect(screen.getByText('Active')).toBeInTheDocument();
  });

  it('applies the variant class', () => {
    render(<Badge label="Active" variant="success" />);

    expect(screen.getByText('Active')).toHaveClass('badge-success');
  });

  it('renders nothing when hidden', () => {
    render(<Badge label="Active" hidden />);

    expect(screen.queryByText('Active')).not.toBeInTheDocument();
  });
});
```

### Component with User Interaction

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

describe('Counter', () => {
  it('starts at the initial count', () => {
    render(<Counter initialCount={5} />);

    expect(screen.getByRole('status')).toHaveTextContent('5');
  });

  it('increments when the increment button is clicked', async () => {
    const user = userEvent.setup();
    render(<Counter initialCount={0} />);

    await user.click(screen.getByRole('button', { name: /increment/i }));

    expect(screen.getByRole('status')).toHaveTextContent('1');
  });

  it('calls onChange with the new count', async () => {
    const user = userEvent.setup();
    const onChange = vi.fn();
    render(<Counter initialCount={0} onChange={onChange} />);

    await user.click(screen.getByRole('button', { name: /increment/i }));

    expect(onChange).toHaveBeenCalledWith(1);
  });
});
```

### Form Component Test

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('submits the form with email and password', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'user@example.com');
    await user.type(screen.getByLabelText(/password/i), 'secret123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'secret123',
    });
  });

  it('shows a validation error for empty email', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByRole('alert')).toHaveTextContent(/email is required/i);
  });

  it('disables the submit button while submitting', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn(() => new Promise(() => {})); // never resolves
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'user@example.com');
    await user.type(screen.getByLabelText(/password/i), 'secret123');
    await user.click(screen.getByRole('button', { name: /sign in/i }));

    expect(screen.getByRole('button', { name: /sign in/i })).toBeDisabled();
  });
});
```

## Query Priority

Use queries in this priority order. Higher priority = more accessible, more resilient to implementation changes:

| Priority | Query | When to use |
|---|---|---|
| 1 | `getByRole` | Buttons, links, headings, form controls, regions — anything with an ARIA role |
| 2 | `getByLabelText` | Form inputs with associated labels |
| 3 | `getByPlaceholderText` | Inputs without visible labels (rare, prefer labels) |
| 4 | `getByText` | Non-interactive text content |
| 5 | `getByDisplayValue` | Current value of form elements |
| 6 | `getByAltText` | Images |
| 7 | `getByTitle` | Elements with title attributes |
| 8 | `getByTestId` | Last resort — when no accessible query works |

### Query Variants

| Variant | Returns | Throws? | Use when |
|---|---|---|---|
| `getBy*` | Element | Yes, if not found | Element should be present |
| `queryBy*` | Element or `null` | No | Asserting element is NOT present |
| `findBy*` | Promise\<Element\> | Yes, if not found after timeout | Element appears asynchronously |

```typescript
// Element IS present
expect(screen.getByRole('button', { name: /save/i })).toBeInTheDocument();

// Element is NOT present
expect(screen.queryByRole('alert')).not.toBeInTheDocument();

// Element appears after async operation
expect(await screen.findByRole('alert')).toHaveTextContent('Saved');
```

## Hook Testing

### Basic Hook Test

```typescript
import { describe, it, expect } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useToggle } from './use-toggle';

describe('useToggle', () => {
  it('starts with the initial value', () => {
    const { result } = renderHook(() => useToggle(false));

    expect(result.current.value).toBe(false);
  });

  it('toggles the value', () => {
    const { result } = renderHook(() => useToggle(false));

    act(() => {
      result.current.toggle();
    });

    expect(result.current.value).toBe(true);
  });

  it('sets to true', () => {
    const { result } = renderHook(() => useToggle(false));

    act(() => {
      result.current.setTrue();
    });

    expect(result.current.value).toBe(true);
  });
});
```

### Hook with Dependencies

```typescript
import { describe, it, expect } from 'vitest';
import { renderHook } from '@testing-library/react';
import { useLocalStorage } from './use-local-storage';

describe('useLocalStorage', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('returns the initial value when storage is empty', () => {
    const { result } = renderHook(() => useLocalStorage('key', 'default'));

    expect(result.current[0]).toBe('default');
  });

  it('returns the stored value when storage has data', () => {
    localStorage.setItem('key', JSON.stringify('stored'));

    const { result } = renderHook(() => useLocalStorage('key', 'default'));

    expect(result.current[0]).toBe('stored');
  });

  it('updates storage when the value changes', () => {
    const { result } = renderHook(() => useLocalStorage('key', 'default'));

    act(() => {
      result.current[1]('updated');
    });

    expect(JSON.parse(localStorage.getItem('key')!)).toBe('updated');
  });

  it('responds to key changes', () => {
    const { result, rerender } = renderHook(
      ({ key }) => useLocalStorage(key, 'default'),
      { initialProps: { key: 'key-a' } },
    );

    act(() => {
      result.current[1]('value-a');
    });

    rerender({ key: 'key-b' });

    expect(result.current[0]).toBe('default');
  });
});
```

### Hook Conventions

1. **Use `renderHook`** — never render a dummy component just to test a hook.
2. **Wrap state updates in `act()`** — any call that triggers a state change must be inside `act()`.
3. **Access results via `result.current`** — this always reflects the latest return value.
4. **Use `rerender` to test prop/dependency changes** — pass new props to simulate the hook re-running with different inputs.

## Context / Provider Testing

### Testing a Provider

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { DataGridProvider, useDataGridContext } from './datagrid-context';

function TestConsumer() {
  const { config, setPageSize } = useDataGridContext();
  return (
    <div>
      <span data-testid="page-size">{config.pageSize}</span>
      <button onClick={() => setPageSize(50)}>Set 50</button>
    </div>
  );
}

describe('DataGridProvider', () => {
  it('provides default config values', () => {
    render(
      <DataGridProvider>
        <TestConsumer />
      </DataGridProvider>,
    );

    expect(screen.getByTestId('page-size')).toHaveTextContent('25');
  });

  it('updates page size through context', async () => {
    const user = userEvent.setup();
    render(
      <DataGridProvider>
        <TestConsumer />
      </DataGridProvider>,
    );

    await user.click(screen.getByRole('button', { name: /set 50/i }));

    expect(screen.getByTestId('page-size')).toHaveTextContent('50');
  });

  it('accepts a custom default page size', () => {
    render(
      <DataGridProvider defaultPageSize={10}>
        <TestConsumer />
      </DataGridProvider>,
    );

    expect(screen.getByTestId('page-size')).toHaveTextContent('10');
  });
});
```

### Testing the Consumer Hook Throws Without Provider

```typescript
import { describe, it, expect } from 'vitest';
import { renderHook } from '@testing-library/react';
import { useDataGridContext } from './datagrid-context';

describe('useDataGridContext', () => {
  it('throws when used outside a provider', () => {
    expect(() => {
      renderHook(() => useDataGridContext());
    }).toThrow('useDataGridContext must be used within a <DataGridProvider>');
  });
});
```

## Custom Render Wrapper

When components depend on providers, create a custom render function to avoid wrapping every test:

### src/test-utils.tsx

```typescript
import { render, type RenderOptions } from '@testing-library/react';
import { type ReactElement, type ReactNode } from 'react';
import { DataGridProvider } from './context/datagrid-context';

interface WrapperProps {
  children: ReactNode;
}

function AllProviders({ children }: WrapperProps) {
  return (
    <DataGridProvider>
      {children}
    </DataGridProvider>
  );
}

function renderWithProviders(ui: ReactElement, options?: Omit<RenderOptions, 'wrapper'>) {
  return render(ui, { wrapper: AllProviders, ...options });
}

export { renderWithProviders as render, screen, within, waitFor } from '@testing-library/react';
```

Then import from test-utils instead of `@testing-library/react`:

```typescript
import { render, screen } from '../test-utils';
```

## Async Testing

### Waiting for State Updates

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('loads and displays user data', async () => {
    render(<UserProfile userId={1} />);

    // Loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument();

    // Wait for data to appear
    expect(await screen.findByRole('heading', { name: /alice/i })).toBeInTheDocument();

    // Loading state should be gone
    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });
});
```

### Waiting for Disappearance

```typescript
it('hides the modal after saving', async () => {
  const user = userEvent.setup();
  render(<EditModal isOpen />);

  await user.click(screen.getByRole('button', { name: /save/i }));

  await waitFor(() => {
    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });
});
```

## Mocking in React Tests

### Mocking a Module Used by a Component

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { UserProfile } from './UserProfile';

vi.mock('./api/users', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: 1, name: 'Alice' }),
}));

describe('UserProfile', () => {
  it('renders the user name', async () => {
    render(<UserProfile userId={1} />);

    expect(await screen.findByText('Alice')).toBeInTheDocument();
  });
});
```

### Mocking Inertia's usePage

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { AssetIndex } from './AssetIndex';

vi.mock('@inertiajs/react', () => ({
  usePage: vi.fn().mockReturnValue({
    props: {
      auth: { user: { name: 'Alice' } },
      assets: { data: [], meta: { total: 0 } },
      can: { create: true },
    },
  }),
  Link: ({ children, ...props }: any) => <a {...props}>{children}</a>,
  router: {
    visit: vi.fn(),
    reload: vi.fn(),
  },
}));
```

### Mocking Inertia's router

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { router } from '@inertiajs/react';
import { DeleteButton } from './DeleteButton';

vi.mock('@inertiajs/react', async () => {
  const actual = await vi.importActual('@inertiajs/react');
  return {
    ...actual,
    router: {
      delete: vi.fn(),
    },
  };
});

describe('DeleteButton', () => {
  it('navigates via Inertia router on confirm', async () => {
    const user = userEvent.setup();
    render(<DeleteButton assetId={1} />);

    await user.click(screen.getByRole('button', { name: /delete/i }));
    await user.click(screen.getByRole('button', { name: /confirm/i }));

    expect(router.delete).toHaveBeenCalledWith('/assets/1');
  });
});
```

## Accessibility Assertions

```typescript
it('has an accessible name', () => {
  render(<SearchInput />);

  expect(screen.getByRole('searchbox')).toHaveAccessibleName('Search assets');
});

it('announces errors to screen readers', async () => {
  const user = userEvent.setup();
  render(<LoginForm onSubmit={vi.fn()} />);

  await user.click(screen.getByRole('button', { name: /sign in/i }));

  const alert = screen.getByRole('alert');
  expect(alert).toBeVisible();
  expect(alert).toHaveTextContent(/email is required/i);
});

it('manages focus on modal open', async () => {
  const user = userEvent.setup();
  render(<ModalTrigger />);

  await user.click(screen.getByRole('button', { name: /open/i }));

  expect(screen.getByRole('dialog')).toHaveFocus();
});
```

## Testing shadcn & Radix Components

shadcn components have specific testing considerations due to CVA variants, Radix primitive behavior, compound component composition, and the `asChild`/Slot pattern.

### Testing CVA Variant Output

Verify that each variant applies the correct classes. Import the CVA function directly — don't render the component just to check classes:

```typescript
import { describe, it, expect } from 'vitest';
import { buttonVariants } from './button';

describe('buttonVariants', () => {
  it('applies default variant classes', () => {
    const classes = buttonVariants();

    expect(classes).toContain('bg-primary');
    expect(classes).toContain('h-9');
  });

  it('applies destructive variant classes', () => {
    const classes = buttonVariants({ variant: 'destructive' });

    expect(classes).toContain('bg-destructive');
  });

  it('applies size classes', () => {
    const classes = buttonVariants({ size: 'sm' });

    expect(classes).toContain('h-8');
    expect(classes).toContain('text-xs');
  });

  it('merges custom className', () => {
    const classes = buttonVariants({ className: 'custom-class' });

    expect(classes).toContain('custom-class');
  });
});
```

Test the CVA function as a unit (belongs in `ts-test` territory). Test the rendered component's behavior separately (belongs here).

### Testing Radix Primitive Behavior

Radix primitives handle accessibility, keyboard navigation, and state management. Test the user-facing behavior, not the Radix internals:

#### Open/Close State

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Select, SelectTrigger, SelectContent, SelectItem, SelectValue } from './select';

describe('Select', () => {
  it('opens the dropdown when the trigger is clicked', async () => {
    const user = userEvent.setup();
    render(
      <Select>
        <SelectTrigger>
          <SelectValue placeholder="Pick one" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="a">Option A</SelectItem>
          <SelectItem value="b">Option B</SelectItem>
        </SelectContent>
      </Select>,
    );

    // Content not visible initially
    expect(screen.queryByRole('option')).not.toBeInTheDocument();

    // Open
    await user.click(screen.getByRole('combobox'));

    // Options visible
    expect(screen.getByRole('option', { name: 'Option A' })).toBeVisible();
    expect(screen.getByRole('option', { name: 'Option B' })).toBeVisible();
  });

  it('selects an option and closes', async () => {
    const user = userEvent.setup();
    const onValueChange = vi.fn();
    render(
      <Select onValueChange={onValueChange}>
        <SelectTrigger>
          <SelectValue placeholder="Pick one" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="a">Option A</SelectItem>
        </SelectContent>
      </Select>,
    );

    await user.click(screen.getByRole('combobox'));
    await user.click(screen.getByRole('option', { name: 'Option A' }));

    expect(onValueChange).toHaveBeenCalledWith('a');
  });
});
```

#### Keyboard Navigation

```typescript
it('supports keyboard navigation', async () => {
  const user = userEvent.setup();
  render(
    <Select>
      <SelectTrigger>
        <SelectValue placeholder="Pick one" />
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="a">Option A</SelectItem>
        <SelectItem value="b">Option B</SelectItem>
      </SelectContent>
    </Select>,
  );

  // Open with Enter
  screen.getByRole('combobox').focus();
  await user.keyboard('{Enter}');

  // Navigate with arrow keys
  await user.keyboard('{ArrowDown}');

  // Select with Enter
  await user.keyboard('{Enter}');
});
```

#### Dialog / Modal

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Dialog, DialogTrigger, DialogContent, DialogTitle } from './dialog';

describe('Dialog', () => {
  it('opens when the trigger is clicked', async () => {
    const user = userEvent.setup();
    render(
      <Dialog>
        <DialogTrigger>Open</DialogTrigger>
        <DialogContent>
          <DialogTitle>My Dialog</DialogTitle>
          <p>Content here</p>
        </DialogContent>
      </Dialog>,
    );

    await user.click(screen.getByRole('button', { name: 'Open' }));

    expect(screen.getByRole('dialog')).toBeVisible();
    expect(screen.getByText('My Dialog')).toBeInTheDocument();
  });

  it('closes on Escape', async () => {
    const user = userEvent.setup();
    render(
      <Dialog defaultOpen>
        <DialogContent>
          <DialogTitle>My Dialog</DialogTitle>
        </DialogContent>
      </Dialog>,
    );

    expect(screen.getByRole('dialog')).toBeVisible();

    await user.keyboard('{Escape}');

    await waitFor(() => {
      expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
    });
  });

  it('traps focus inside the dialog', async () => {
    const user = userEvent.setup();
    render(
      <Dialog defaultOpen>
        <DialogContent>
          <DialogTitle>My Dialog</DialogTitle>
          <button>First</button>
          <button>Second</button>
        </DialogContent>
      </Dialog>,
    );

    // Tab cycles through focusable elements inside the dialog
    await user.tab();
    expect(screen.getByRole('button', { name: 'First' })).toHaveFocus();

    await user.tab();
    expect(screen.getByRole('button', { name: 'Second' })).toHaveFocus();
  });
});
```

### Testing Compound Components

Test compound components as an assembled unit — the way a consumer uses them:

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from './card';

describe('Card', () => {
  it('renders all compound parts', () => {
    render(
      <Card>
        <CardHeader>
          <CardTitle>Title</CardTitle>
          <CardDescription>Description</CardDescription>
        </CardHeader>
        <CardContent>Body content</CardContent>
        <CardFooter>Footer content</CardFooter>
      </Card>,
    );

    expect(screen.getByText('Title')).toBeInTheDocument();
    expect(screen.getByText('Description')).toBeInTheDocument();
    expect(screen.getByText('Body content')).toBeInTheDocument();
    expect(screen.getByText('Footer content')).toBeInTheDocument();
  });

  it('passes through className to each part', () => {
    const { container } = render(
      <Card className="custom-card">
        <CardHeader className="custom-header">
          <CardTitle>Title</CardTitle>
        </CardHeader>
      </Card>,
    );

    expect(container.querySelector('[data-slot="card"]')).toHaveClass('custom-card');
    expect(container.querySelector('[data-slot="card-header"]')).toHaveClass('custom-header');
  });

  it('spreads additional props', () => {
    render(
      <Card data-testid="my-card" aria-label="Asset card">
        <CardContent>Content</CardContent>
      </Card>,
    );

    expect(screen.getByTestId('my-card')).toHaveAttribute('aria-label', 'Asset card');
  });
});
```

### Testing the `asChild` / Slot Pattern

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { Button } from './button';

describe('Button asChild', () => {
  it('renders as a button by default', () => {
    render(<Button>Click</Button>);

    expect(screen.getByRole('button', { name: 'Click' })).toBeInTheDocument();
  });

  it('renders as the child element when asChild is true', () => {
    render(
      <Button asChild>
        <a href="/about">About</a>
      </Button>,
    );

    const link = screen.getByRole('link', { name: 'About' });
    expect(link).toBeInTheDocument();
    expect(link).toHaveAttribute('href', '/about');
    // Should NOT render a button element
    expect(screen.queryByRole('button')).not.toBeInTheDocument();
  });

  it('merges className onto the child element', () => {
    render(
      <Button asChild variant="outline" className="extra">
        <a href="/about">About</a>
      </Button>,
    );

    const link = screen.getByRole('link', { name: 'About' });
    expect(link).toHaveClass('extra');
  });
});
```

### Testing Context-Dependent Components

For components that require a provider (Sidebar, DataGrid, etc.), create a wrapper:

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SidebarProvider, Sidebar, SidebarTrigger } from './sidebar';

function renderSidebar(props = {}) {
  return render(
    <SidebarProvider {...props}>
      <SidebarTrigger aria-label="Toggle sidebar" />
      <Sidebar>
        <nav>Sidebar content</nav>
      </Sidebar>
    </SidebarProvider>,
  );
}

describe('Sidebar', () => {
  it('starts expanded by default', () => {
    renderSidebar();

    expect(screen.getByText('Sidebar content')).toBeVisible();
    expect(screen.getByRole('complementary')).toHaveAttribute('data-state', 'expanded');
  });

  it('collapses when the trigger is clicked', async () => {
    const user = userEvent.setup();
    renderSidebar();

    await user.click(screen.getByRole('button', { name: /toggle sidebar/i }));

    expect(screen.getByRole('complementary')).toHaveAttribute('data-state', 'collapsed');
  });

  it('starts collapsed when defaultOpen is false', () => {
    renderSidebar({ defaultOpen: false });

    expect(screen.getByRole('complementary')).toHaveAttribute('data-state', 'collapsed');
  });
});
```

### Querying by `data-slot`

When a component part has no accessible role, use `data-slot` as a fallback query:

```typescript
import { within } from '@testing-library/react';

it('renders content inside the card body', () => {
  const { container } = render(
    <Card>
      <CardContent>Important text</CardContent>
    </Card>,
  );

  const content = container.querySelector('[data-slot="card-content"]')!;
  expect(within(content).getByText('Important text')).toBeInTheDocument();
});
```

**Prefer role queries first.** Only fall back to `data-slot` when the element has no semantic role and adding one would be incorrect.

### Radix Portal Gotchas

Radix renders some content (Select dropdown, Dialog, Popover) into a portal outside the component tree. This means:

1. **`screen` queries still work** — Testing Library queries the entire document, not just the render container.
2. **`container.querySelector` won't find portal content** — use `screen` queries or `document.querySelector` instead.
3. **Animation states** — Radix uses `data-state="open"` / `data-state="closed"` with CSS animations. Use `waitFor` when asserting disappearance after close:

```typescript
// Wrong — element may still be animating out
expect(screen.queryByRole('dialog')).not.toBeInTheDocument();

// Right — wait for animation to complete
await waitFor(() => {
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
});
```

## What This Skill Does NOT Cover

- **Pure TypeScript unit tests** (no rendering) → use `ts-test` skill
- **CVA variant function unit tests** (no rendering) → use `ts-test` skill
- **Creating shadcn components** → use `shadcn-component` skill
- **Inertia page/layout creation** → use `ts-inertia` skill
- **End-to-end browser testing** → use Playwright (manual)
- **Visual regression testing** → use Storybook + Chromatic (manual)

## Checklist

When creating React tests:

1. **Read the component/hook** — understand props, state, context dependencies, and user interactions
2. **Identify providers needed** — does the component require context providers? Create a wrapper or use `renderWithProviders`
3. **Set up `userEvent`** — `const user = userEvent.setup()` at the start of interaction tests
4. **Write the happy path** — render with valid props, assert visible output
5. **Test interactions** — click, type, select — and assert the resulting UI changes
6. **Test error/empty states** — missing data, validation errors, loading states
7. **Test accessibility** — query by role, assert accessible names, check focus management
8. **Use `findBy*` for async** — don't use `getBy*` + `waitFor` when `findBy*` does the job
9. **Run the tests** — `npm run test` or `npx vitest run {file}` to verify they pass
