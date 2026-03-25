---
name: ts-package
description: Create TypeScript packages with hooks, context providers, components, and build configuration. Use when the user wants to scaffold or extend a frontend library.
argument-hint: "[package name or domain context]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, WebSearch
---

# TypeScript Package Skill

You help create TypeScript packages — standalone libraries that bundle reusable functionality like custom hooks, context providers, utility functions, and components. Packages are built with tsup, tested with Vitest, and published as dual CJS/ESM modules with full type declarations.

## Core Principles

1. **One package, one concern** — a package solves a specific problem. If it's doing two unrelated things, it's two packages.
2. **Peer dependencies for frameworks** — React, React DOM, and other framework packages are always `peerDependencies`, never bundled.
3. **Barrel exports control the public API** — `src/index.ts` is the single entry point. If it's not exported from the barrel, it's not public.
4. **Types are first-class** — every export has explicit TypeScript types. Declaration files are generated alongside the build output.
5. **Tests live next to code** — test files are collocated with the modules they test, not in a separate tree.

## Discovery

Before creating a package, scan the project to understand what exists:

1. **Find existing packages** — `Glob` for `**/package.json` to discover sibling packages and their conventions.
2. **Find existing hooks** — `Grep` for `export function use` or `export const use` to discover custom hooks already in the project.
3. **Find existing context providers** — `Grep` for `createContext` to discover provider patterns.
4. **Find build configuration** — `Glob` for `**/tsup.config.ts` and `**/tsconfig.json` to match existing conventions.
5. **Find test configuration** — `Glob` for `**/vitest.config.ts` or `**/jest.config.ts` to match existing test setup.

## Package Structure

```
{package-name}/
├── src/
│   ├── index.ts                 # Barrel exports — the public API
│   ├── hooks/
│   │   ├── use-{name}.ts        # Custom hook implementation
│   │   └── use-{name}.test.ts   # Collocated test
│   ├── context/
│   │   ├── {name}-context.ts    # Context definition + provider + consumer hook
│   │   └── {name}-context.test.tsx
│   ├── components/
│   │   ├── {Name}.tsx           # Component implementation
│   │   └── {Name}.test.tsx      # Collocated test
│   └── utils/
│       ├── {name}.ts            # Utility function
│       └── {name}.test.ts       # Collocated test
├── package.json
├── tsconfig.json
├── tsup.config.ts
├── vitest.config.ts
└── README.md
```

### File Naming Conventions

| Type | File name | Export name |
|---|---|---|
| Hook | `use-local-storage.ts` | `useLocalStorage` |
| Context | `datagrid-context.ts` | `DataGridContext`, `DataGridProvider`, `useDataGridContext` |
| Component | `FilterPanel.tsx` | `FilterPanel` |
| Utility | `merge-classes.ts` | `mergeClasses` |
| Type | exported from the module that uses it | `DataGridConfig`, `UseLocalStorageOptions` |

Files use kebab-case. Exports use camelCase (hooks/utils) or PascalCase (components/contexts/types).

## Scaffold Templates

### package.json

```json
{
  "name": "@{scope}/{package-name}",
  "version": "0.0.1",
  "description": "{One-line description}",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.js"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  },
  "files": [
    "dist"
  ],
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "prepublishOnly": "npm run build"
  },
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  },
  "devDependencies": {
    "@testing-library/react": "^16.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/user-event": "^14.0.0",
    "jsdom": "^25.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "tsup": "^8.0.0",
    "typescript": "^5.0.0",
    "vitest": "^3.0.0"
  }
}
```

**Notes:**
- `peerDependencies` declares React as a peer — the consuming app provides it.
- React is also in `devDependencies` so tests can run standalone.
- Omit `peerDependencies` for React if the package has no React dependency (pure TS utility library).
- Adjust scope (`@{scope}/`) to match the user's npm org or remove for unscoped packages.

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.test.tsx"]
}
```

### tsup.config.ts

```typescript
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  sourcemap: true,
  clean: true,
  external: ['react', 'react-dom'],
  shims: true,
});
```

**Notes:**
- `external` must list all `peerDependencies` to avoid bundling them.
- `dts: true` generates `.d.ts` declaration files alongside the output.
- Add additional entry points to `entry` array if the package has multiple subpath exports.

### vitest.config.ts

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

Use `environment: 'node'` for pure utility packages with no DOM dependency.

### src/test-setup.ts

```typescript
import '@testing-library/jest-dom/vitest';
```

Only needed for packages with React/DOM code.

### src/index.ts — Barrel Exports

```typescript
// Hooks
export { useLocalStorage } from './hooks/use-local-storage';
export type { UseLocalStorageOptions } from './hooks/use-local-storage';

// Context
export { DataGridProvider, useDataGridContext } from './context/datagrid-context';
export type { DataGridConfig } from './context/datagrid-context';

// Components
export { FilterPanel } from './components/FilterPanel';
export type { FilterPanelProps } from './components/FilterPanel';

// Utilities
export { mergeClasses } from './utils/merge-classes';
```

**Rules for barrel exports:**
- Export only the public API — internal helpers stay internal.
- Always export associated types alongside their implementations.
- Use named exports exclusively — no default exports from a library.
- Group exports by category with comments.

## Custom Hook Patterns

### Basic Hook

```typescript
import { useState, useCallback } from 'react';

export interface UseToggleReturn {
  value: boolean;
  toggle: () => void;
  setTrue: () => void;
  setFalse: () => void;
}

export function useToggle(initialValue = false): UseToggleReturn {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue((v) => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse };
}
```

### Generic Hook with Options

For hooks with configuration, accept a single options object. Use generic type parameters on the function. Return a tuple `[value, setter]` or a named interface for object returns.

### Hook with Cleanup

For hooks with side effects, store the latest callback in a `useRef` and clean up in the `useEffect` return.

### Hook Conventions

1. **Return type is explicit** — define an interface for object returns, a tuple type for array returns.
2. **Options are a single object** — not multiple optional parameters.
3. **Use `useCallback` for returned functions** — prevents unnecessary re-renders in consumers.
4. **Use `useRef` for mutable values that shouldn't trigger re-renders.**
5. **Generic type parameters go on the function** — `useLocalStorage<T>(key, initialValue)`.

## Context + Provider Pattern

### Template

Each context file contains three parts: context creation, provider component, and consumer hook.

```typescript
const XContext = createContext<XContextValue | null>(null);

export function XProvider({ children, ...config }: XProviderProps) {
  const [state, setState] = useState(/* defaults */);
  const value = useMemo(() => ({ state, ...actions }), [state]);
  return <XContext.Provider value={value}>{children}</XContext.Provider>;
}

export function useX(): XContextValue {
  const context = useContext(XContext);
  if (!context) throw new Error('useX must be used within <XProvider>');
  return context;
}
```

### Context Conventions

1. **Initialize with `null`, throw if null** — catches missing providers at runtime.
2. **Named exports only** — `XProvider`, not default export. Never export the raw context object.
3. **Memoize the context value** — `useMemo` prevents unnecessary re-renders of all consumers.
4. **One file per context** — context definition, provider, and consumer hook live together.

## Component Patterns for Libraries

### Composable Component

```typescript
import { forwardRef, type HTMLAttributes } from 'react';

export interface FilterPanelProps extends HTMLAttributes<HTMLDivElement> {
  isOpen: boolean;
  onClose: () => void;
}

export const FilterPanel = forwardRef<HTMLDivElement, FilterPanelProps>(
  ({ isOpen, onClose, children, className, ...props }, ref) => {
    if (!isOpen) return null;

    return (
      <div ref={ref} className={className} role="region" aria-label="Filters" {...props}>
        {children}
        <button type="button" onClick={onClose}>
          Close
        </button>
      </div>
    );
  },
);

FilterPanel.displayName = 'FilterPanel';
```

### Component Conventions

1. **Extend native HTML attributes** — `extends HTMLAttributes<HTMLDivElement>` lets consumers pass standard DOM props.
2. **Forward refs** — library components should always forward refs so consumers can access the underlying DOM node.
3. **Spread remaining props** — `{...props}` on the root element.
4. **No hardcoded styles** — accept `className` and let the consumer style. Tailwind classes are fine for defaults if documented.
5. **Accessibility built in** — `role`, `aria-*` attributes, keyboard handling.
6. **`displayName` on forwardRef components** — required for React DevTools.

## Adding to an Existing Package

When adding new functionality to an existing package:

1. **Read the barrel export** (`src/index.ts`) to understand the current public API.
2. **Follow the existing file structure** — don't introduce new directories without reason.
3. **Add the export to the barrel** after creating the implementation.
4. **Match existing patterns** — if other hooks return objects, don't return a tuple. If other components use forwardRef, use forwardRef.

## Pure Utility Package (No React)

For packages without React dependencies:

1. **Remove React from `peerDependencies`** and `devDependencies`.
2. **Remove `jsx` from tsconfig** — set `"jsx": undefined` or remove the field.
3. **Remove `external: ['react', 'react-dom']`** from tsup config.
4. **Use `environment: 'node'`** in vitest config (unless the utility needs DOM APIs, then keep `jsdom`).
5. **Omit the test setup file** — no `@testing-library/jest-dom` needed.

## Checklist

When creating a TypeScript package:

1. **Scaffold the project** — package.json, tsconfig.json, tsup.config.ts, vitest.config.ts
2. **Determine if React is needed** — adjust peer dependencies and tsup externals accordingly
3. **Create the implementation** — hooks, context, components, or utilities in `src/`
4. **Create the barrel export** — `src/index.ts` with only the public API
5. **Write collocated tests** — every module has a `.test.ts(x)` file next to it
6. **Verify the build** — `npm run build` produces `dist/` with `.js`, `.cjs`, `.d.ts`, `.d.cts`
7. **Verify types** — `npm run typecheck` passes with no errors
8. **Verify tests** — `npm run test` passes
