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

### Hook Conventions

1. **Use `renderHook`** — never render a dummy component just to test a hook.
2. **Wrap state updates in `act()`** — any call that triggers a state change must be inside `act()`.
3. **Access results via `result.current`** — this always reflects the latest return value.
4. **Use `rerender` to test prop/dependency changes** — pass new props to simulate the hook re-running with different inputs.

## Context / Provider Testing

Create a `TestConsumer` component that reads from context, then test the provider's behavior through it:

```typescript
function TestConsumer() {
  const { config, setPageSize } = useDataGridContext();
  return (
    <div>
      <span data-testid="page-size">{config.pageSize}</span>
      <button onClick={() => setPageSize(50)}>Set 50</button>
    </div>
  );
}
```

Test patterns for providers:
- **Default values** — render with provider, assert defaults appear
- **State updates** — interact with consumer, assert context updates
- **Custom initial values** — pass props to provider, assert consumer reads them
- **Missing provider** — `renderHook(() => useContext())` should throw

### Custom Render Wrapper

When components depend on providers, create `src/test-utils.tsx` that wraps `render` with all needed providers:

```typescript
import { render, type RenderOptions } from '@testing-library/react';
import { type ReactElement, type ReactNode } from 'react';

function AllProviders({ children }: { children: ReactNode }) {
  return <DataGridProvider>{children}</DataGridProvider>;
}

export function renderWithProviders(ui: ReactElement, options?: Omit<RenderOptions, 'wrapper'>) {
  return render(ui, { wrapper: AllProviders, ...options });
}
```

## Async Testing

```typescript
// Use findBy* for elements that appear after async operations
expect(await screen.findByRole('heading', { name: /alice/i })).toBeInTheDocument();

// Use waitFor for disappearance or complex async assertions
await waitFor(() => {
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
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

Use `vi.importActual` to preserve real exports while overriding `router`:

```typescript
vi.mock('@inertiajs/react', async () => {
  const actual = await vi.importActual('@inertiajs/react');
  return { ...actual, router: { delete: vi.fn() } };
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

Test the user-facing behavior, not Radix internals. Key patterns:

- **Open/close** — click trigger, assert content visible/hidden via roles (`combobox`, `option`, `dialog`)
- **Selection** — click option, assert `onValueChange` called with correct value
- **Keyboard** — focus trigger, `user.keyboard('{Enter}')` to open, `{ArrowDown}` to navigate, `{Enter}` to select
- **Dialog close** — `user.keyboard('{Escape}')`, use `waitFor` to assert disappearance (animation delay)
- **Focus trapping** — `user.tab()` inside dialog, assert focus stays within

### Testing Compound Components

Test as an assembled unit. Assert all parts render, `className` passes through to `data-slot` elements, and additional props spread correctly.

### Testing `asChild` / Slot

Three cases: renders as default element, renders as child element when `asChild={true}` (assert role changes, e.g., button → link), and merges className onto child.

### Testing Context-Dependent Components

Create a local `renderWithProvider()` helper that wraps the component in its required provider. Test default state, state changes via interaction, and custom initial props.

### Querying by `data-slot`

When a component part has no accessible role, fall back to `container.querySelector('[data-slot="card-content"]')` with `within()`. **Prefer role queries first.**

### Radix Portal Gotchas

1. **`screen` queries still work** — Testing Library queries the entire document, not just the render container.
2. **`container.querySelector` won't find portal content** — use `screen` queries instead.
3. **Animation states** — use `waitFor` when asserting disappearance after close (Radix animates with `data-state`).

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
