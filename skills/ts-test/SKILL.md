---
name: ts-test
description: Create TypeScript unit tests with Vitest or Jest. Use when the user wants to add, modify, or fix tests for non-React TypeScript code.
argument-hint: "[module or function to test]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# TypeScript Test Skill

You help create unit tests for TypeScript code using Vitest (preferred) or Jest. This skill covers testing pure functions, classes, utilities, async operations, and mocking — everything except React component/hook testing, which belongs in the `react-test` skill.

## Core Principles

1. **Test behavior, not implementation** — assert what the code does, not how it does it. If a refactor doesn't change behavior, tests shouldn't break.
2. **One assertion concept per test** — each `it` block tests one logical thing. Multiple `expect` calls are fine if they verify the same concept.
3. **Tests are documentation** — `describe` and `it` strings should read as a specification. A reader should understand the module's behavior from the test file alone.
4. **Collocated by default** — test files live next to the module they test: `use-toggle.ts` → `use-toggle.test.ts`.
5. **No test logic** — tests should not contain conditionals, loops, or complex setup. If you need shared setup, use `beforeEach`. If a test is hard to write, the code under test may need a better interface.

## Discovery

Before writing tests:

1. **Find the test runner** — `Glob` for `**/vitest.config.ts` or `**/jest.config.ts` to determine which runner is in use.
2. **Find existing tests** — `Glob` for `**/*.test.ts` to discover naming and structural conventions.
3. **Read the module under test** — understand its exports, types, and edge cases before writing any test.
4. **Check for test utilities** — `Grep` for `test-utils` or `test-helpers` to find shared setup code.

## Vitest Configuration

### vitest.config.ts (Node environment)

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    environment: 'node',
    globals: true,
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### vitest.config.ts (DOM environment)

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test-setup.ts'],
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

Use `node` for pure logic. Use `jsdom` only when the code under test interacts with DOM APIs (localStorage, window, document).

## Test File Structure

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { calculateTotal } from './calculate-total';

describe('calculateTotal', () => {
  it('returns 0 for an empty list', () => {
    expect(calculateTotal([])).toBe(0);
  });

  it('sums item prices', () => {
    const items = [
      { name: 'Widget', price: 10 },
      { name: 'Gadget', price: 25 },
    ];

    expect(calculateTotal(items)).toBe(35);
  });

  it('applies a discount percentage', () => {
    const items = [{ name: 'Widget', price: 100 }];

    expect(calculateTotal(items, { discount: 0.1 })).toBe(90);
  });

  it('throws on negative prices', () => {
    const items = [{ name: 'Widget', price: -5 }];

    expect(() => calculateTotal(items)).toThrow('Price cannot be negative');
  });
});
```

### Structure Conventions

1. **One `describe` per export** — matches the module's public API surface.
2. **`it` strings start with a verb** — "returns", "throws", "calls", "sets", "emits".
3. **Arrange → Act → Assert** — blank lines separate the three phases when the test is more than a one-liner.
4. **No nesting beyond two levels** — `describe` > `it`. Use a second `describe` inside for method grouping on classes, not for conditional branching.

## Testing Patterns

### Pure Functions

```typescript
import { describe, it, expect } from 'vitest';
import { slugify } from './slugify';

describe('slugify', () => {
  it('converts spaces to hyphens', () => {
    expect(slugify('hello world')).toBe('hello-world');
  });

  it('lowercases the string', () => {
    expect(slugify('Hello World')).toBe('hello-world');
  });

  it('strips non-alphanumeric characters', () => {
    expect(slugify('Hello, World!')).toBe('hello-world');
  });

  it('handles empty strings', () => {
    expect(slugify('')).toBe('');
  });
});
```

### Classes

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { EventEmitter } from './event-emitter';

describe('EventEmitter', () => {
  let emitter: EventEmitter;

  beforeEach(() => {
    emitter = new EventEmitter();
  });

  describe('on', () => {
    it('registers a listener', () => {
      const listener = vi.fn();
      emitter.on('test', listener);
      emitter.emit('test');

      expect(listener).toHaveBeenCalledOnce();
    });

    it('passes event data to the listener', () => {
      const listener = vi.fn();
      emitter.on('test', listener);
      emitter.emit('test', { id: 1 });

      expect(listener).toHaveBeenCalledWith({ id: 1 });
    });
  });

  describe('off', () => {
    it('removes a listener', () => {
      const listener = vi.fn();
      emitter.on('test', listener);
      emitter.off('test', listener);
      emitter.emit('test');

      expect(listener).not.toHaveBeenCalled();
    });
  });
});
```

### Async Functions

```typescript
import { describe, it, expect, vi } from 'vitest';
import { fetchUser } from './fetch-user';

describe('fetchUser', () => {
  it('returns the user data', async () => {
    const user = await fetchUser(1);

    expect(user).toEqual({
      id: 1,
      name: 'Alice',
    });
  });

  it('throws on non-existent user', async () => {
    await expect(fetchUser(999)).rejects.toThrow('User not found');
  });
});
```

### Data-Driven Tests

Use `it.each` when the same logic is tested with multiple inputs:

```typescript
import { describe, it, expect } from 'vitest';
import { isValidEmail } from './is-valid-email';

describe('isValidEmail', () => {
  it.each([
    ['user@example.com', true],
    ['user@sub.example.com', true],
    ['user+tag@example.com', true],
    ['', false],
    ['not-an-email', false],
    ['@example.com', false],
    ['user@', false],
  ])('returns %s for "%s"', (input, expected) => {
    expect(isValidEmail(input)).toBe(expected);
  });
});
```

Use `describe.each` when the same group of tests applies to multiple configurations:

```typescript
describe.each([
  { env: 'development', debug: true },
  { env: 'production', debug: false },
])('in $env environment', ({ env, debug }) => {
  it(`sets debug to ${debug}`, () => {
    const config = createConfig({ environment: env });
    expect(config.debug).toBe(debug);
  });
});
```

### Error and Exception Testing

```typescript
describe('parseConfig', () => {
  it('throws on invalid JSON', () => {
    expect(() => parseConfig('not json')).toThrow(SyntaxError);
  });

  it('throws with a descriptive message for missing required fields', () => {
    expect(() => parseConfig('{}')).toThrow('Missing required field: name');
  });

  it('rejects invalid config asynchronously', async () => {
    await expect(loadConfig('/bad/path')).rejects.toThrow('File not found');
  });
});
```

## Mocking

### Function Mocks

```typescript
import { describe, it, expect, vi } from 'vitest';

it('calls the callback with each item', () => {
  const callback = vi.fn();
  const items = ['a', 'b', 'c'];

  processItems(items, callback);

  expect(callback).toHaveBeenCalledTimes(3);
  expect(callback).toHaveBeenNthCalledWith(1, 'a');
  expect(callback).toHaveBeenNthCalledWith(2, 'b');
  expect(callback).toHaveBeenNthCalledWith(3, 'c');
});
```

### Module Mocks

```typescript
import { describe, it, expect, vi } from 'vitest';
import { sendNotification } from './send-notification';

// Mock an entire module
vi.mock('./email-client', () => ({
  sendEmail: vi.fn().mockResolvedValue({ id: 'msg-123' }),
}));

import { sendEmail } from './email-client';

describe('sendNotification', () => {
  it('sends an email with the correct subject', async () => {
    await sendNotification({ to: 'user@example.com', message: 'Hello' });

    expect(sendEmail).toHaveBeenCalledWith(
      expect.objectContaining({
        to: 'user@example.com',
        subject: expect.stringContaining('Notification'),
      }),
    );
  });
});
```

### Spying on Methods

```typescript
import { describe, it, expect, vi } from 'vitest';

it('calls storage.setItem', () => {
  const spy = vi.spyOn(Storage.prototype, 'setItem');

  savePreference('theme', 'dark');

  expect(spy).toHaveBeenCalledWith('theme', '"dark"');

  spy.mockRestore();
});
```

### Mocking Conventions

1. **Mock at the boundary** — mock external dependencies (API clients, storage, timers), not internal functions.
2. **Use `vi.fn()` for callbacks** — when you need to verify a function was called correctly.
3. **Use `vi.mock()` for modules** — when you need to replace an entire dependency.
4. **Use `vi.spyOn()` for observation** — when you want to watch a method without replacing it (or replace it temporarily).
5. **Always `mockRestore()` spies** — or use `afterEach(() => vi.restoreAllMocks())` globally.
6. **Prefer `mockResolvedValue` / `mockRejectedValue`** over `mockImplementation` for async mocks.

## Timers

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { debounce } from './debounce';

describe('debounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('delays execution by the specified ms', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 300);

    debounced();
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(300);
    expect(fn).toHaveBeenCalledOnce();
  });

  it('resets the timer on subsequent calls', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 300);

    debounced();
    vi.advanceTimersByTime(200);
    debounced(); // reset
    vi.advanceTimersByTime(200);

    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledOnce();
  });
});
```

## Jest Compatibility

When working with an existing Jest setup, the API is nearly identical. Key differences:

| Vitest | Jest |
|---|---|
| `import { vi } from 'vitest'` | `jest` global |
| `vi.fn()` | `jest.fn()` |
| `vi.mock()` | `jest.mock()` |
| `vi.spyOn()` | `jest.spyOn()` |
| `vi.useFakeTimers()` | `jest.useFakeTimers()` |
| `vi.advanceTimersByTime()` | `jest.advanceTimersByTime()` |
| `toHaveBeenCalledOnce()` | `toHaveBeenCalledTimes(1)` |

If the project uses Jest, follow the same patterns from this skill but substitute the Jest equivalents. Do not migrate a project from Jest to Vitest unless explicitly asked.

## What This Skill Does NOT Cover

- **React component rendering** → use `react-test` skill
- **React hook testing with `renderHook`** → use `react-test` skill
- **User interaction simulation** → use `react-test` skill
- **End-to-end or integration tests** → use Playwright or similar (manual)
- **Laravel/PHP tests** → use the testing patterns in the `action` and `controller` skills

## Checklist

When creating tests:

1. **Read the module under test** — understand all exports, parameters, return types, and error conditions
2. **Identify the test runner** — Vitest or Jest, and match the project's existing configuration
3. **Create the test file** — collocated next to the module, matching the naming convention
4. **Write the happy path first** — the most common, expected usage
5. **Add edge cases** — empty inputs, boundaries, null/undefined, type coercion traps
6. **Add error cases** — invalid inputs, thrown exceptions, rejected promises
7. **Add mocks only at boundaries** — external dependencies, timers, storage, network
8. **Run the tests** — `npm run test` or `npx vitest run {file}` to verify they pass
