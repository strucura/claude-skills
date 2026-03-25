---
name: ts-publish
description: Configure npm publishing, versioning, registry settings, and release workflows for TypeScript packages. Use when the user wants to publish or release a package.
argument-hint: "[package name or publishing concern]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# TypeScript Publish Skill

You help configure TypeScript packages for publishing to npm or private registries. This skill covers package.json metadata, versioning strategy, registry configuration, prepublish validation, and release workflows.

## Core Principles

1. **Build before publish** — the `prepublishOnly` script must build the package. Never publish source TypeScript — only the compiled output.
2. **Ship types** — every published package includes `.d.ts` declaration files. TypeScript consumers get type safety without `@types/*` packages.
3. **Minimal package** — the `files` field in package.json controls what gets published. Only include `dist/` and any required metadata. Never publish `src/`, tests, config files, or `node_modules`.
4. **Semver strictly** — breaking changes bump major, new features bump minor, fixes bump patch. No exceptions.
5. **Validate before publishing** — run build, typecheck, and tests before every publish.

## Discovery

Before configuring publishing:

1. **Read package.json** — check current `name`, `version`, `files`, `exports`, `scripts`, and registry configuration.
2. **Check for .npmrc** — `Glob` for `**/.npmrc` to discover existing registry settings.
3. **Check for .npmignore** — `Glob` for `**/.npmignore` (prefer `files` field over `.npmignore`).
4. **Check for CI/CD config** — `Glob` for `**/.github/workflows/*.yml` to see if automated publishing exists.
5. **Check the registry** — `Bash` with `npm view {package-name} version` to see if the package has been published before.

## Package.json for Publishing

### Required Fields

```json
{
  "name": "@scope/package-name",
  "version": "0.1.0",
  "description": "One-line description of what this package does",
  "author": "Author Name",
  "license": "MIT",
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
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "prepublishOnly": "npm run build"
  }
}
```

### Optional Metadata

Add `repository` (with `directory` for monorepos), `homepage`, `bugs`, `keywords`, and `engines` fields as needed.

### Subpath Exports

For packages with multiple entry points:

```json
{
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
    },
    "./utils": {
      "import": {
        "types": "./dist/utils.d.ts",
        "default": "./dist/utils.js"
      },
      "require": {
        "types": "./dist/utils.d.cts",
        "default": "./dist/utils.cjs"
      }
    }
  }
}
```

Update the tsup config to match:

```typescript
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts', 'src/utils.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  sourcemap: true,
  clean: true,
  external: ['react', 'react-dom'],
});
```

## Peer Dependencies

### When to Use Peer Dependencies

| Dependency type | When to use | Example |
|---|---|---|
| `peerDependencies` | Framework/runtime the consumer provides | `react`, `react-dom`, `vue` |
| `dependencies` | Library the package bundles or needs at runtime | `clsx`, `date-fns` |
| `devDependencies` | Build tools, test tools, types for development | `tsup`, `vitest`, `typescript` |

### Peer Dependency Ranges

```json
{
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  },
  "peerDependenciesMeta": {
    "react-dom": {
      "optional": true
    }
  }
}
```

- Use `||` ranges to support multiple major versions.
- Use `peerDependenciesMeta.optional` for dependencies that enhance but aren't required (e.g., `react-dom` for a hook-only package).

## Versioning

### Semantic Versioning Rules

| Change | Version bump | Example |
|---|---|---|
| Breaking change (removed export, changed API signature, renamed type) | Major (`1.0.0` → `2.0.0`) | Removing a hook, changing return type |
| New feature (new export, new optional parameter) | Minor (`1.0.0` → `1.1.0`) | Adding a new hook |
| Bug fix, documentation, internal refactor | Patch (`1.0.0` → `1.0.1`) | Fixing a hook's cleanup |

### Pre-1.0 Versioning

During initial development (`0.x.y`):
- `0.1.0` — initial release
- `0.2.0` — breaking changes (major bumps to minor during 0.x)
- `0.1.1` — bug fixes

### Version Commands

```bash
# Bump patch (0.1.0 → 0.1.1)
npm version patch

# Bump minor (0.1.0 → 0.2.0)
npm version minor

# Bump major (0.1.0 → 1.0.0)
npm version major

# Set a specific version
npm version 1.0.0-beta.1

# Bump without creating a git tag
npm version patch --no-git-tag-version
```

`npm version` updates `package.json`, creates a git commit, and creates a git tag by default.

## Registry Configuration

### Public npm Registry (Default)

No configuration needed. Publish with:

```bash
npm publish --access public
```

The `--access public` flag is required for scoped packages on their first publish (scoped packages default to private on npm).

### Private npm Registry

Create `.npmrc` in the package root:

```
@scope:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}
```

Or for a self-hosted registry:

```
@scope:registry=https://registry.example.com
//registry.example.com/:_authToken=${NPM_TOKEN}
```

**Never hardcode tokens in `.npmrc`** — use environment variables. Add `.npmrc` to `.gitignore` if it contains any credentials. For CI, set `NPM_TOKEN` as a secret in the CI environment.

### GitHub Packages

```
@scope:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

The scope must match the GitHub org/user name.

## Prepublish Validation

### Validate Script

Add a `prepack` or `prepublishOnly` script that runs all checks:

```json
{
  "scripts": {
    "prepublishOnly": "npm run typecheck && npm run test && npm run build"
  }
}
```

This ensures every publish is type-checked, tested, and built. If any step fails, the publish is aborted.

### Dry Run

Always verify what will be published before actually publishing:

```bash
# See what files will be included in the tarball
npm pack --dry-run

# Create the tarball locally for inspection
npm pack
```

Review the output to ensure:
- `dist/` is included with `.js`, `.cjs`, `.d.ts`, `.d.cts` files
- `src/` is NOT included
- Test files are NOT included
- Config files (`.env`, `.npmrc` with tokens) are NOT included

## Publishing Workflow

### Manual Publishing

```bash
# 1. Ensure clean working directory
git status

# 2. Run all validations
npm run typecheck && npm run test && npm run build

# 3. Verify the package contents
npm pack --dry-run

# 4. Bump the version
npm version patch  # or minor, or major

# 5. Publish
npm publish --access public

# 6. Push the version commit and tag
git push && git push --tags
```

### GitHub Actions Workflow

```yaml
name: Publish

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'  # Use 'https://npm.pkg.github.com' for GitHub Packages

      - run: npm ci
      - run: npm run typecheck
      - run: npm run test
      - run: npm run build
      - run: npm publish --access public  # Omit --access for private registries
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}  # Use secrets.GITHUB_TOKEN for GitHub Packages
```

## Prerelease Versions

For beta/alpha/RC releases:

```bash
# Create a prerelease version
npm version 1.0.0-beta.1

# Publish with a dist-tag so it doesn't become "latest"
npm publish --tag beta --access public
```

Consumers install with:

```bash
npm install @scope/package-name@beta
```

### Dist Tags

| Tag | Purpose |
|---|---|
| `latest` | Default install tag. Points to the latest stable version. |
| `beta` | Pre-release versions. Must be explicitly requested. |
| `next` | Upcoming major version. For early adopters. |

```bash
# Set a dist tag manually
npm dist-tag add @scope/package-name@2.0.0-beta.1 beta

# List all tags
npm dist-tag ls @scope/package-name
```

## Monorepo Publishing

For monorepo packages, ensure `repository.directory` is set:

```json
{
  "repository": {
    "type": "git",
    "url": "https://github.com/org/repo.git",
    "directory": "packages/package-name"
  }
}
```

Each package in a monorepo has its own `package.json`, `tsup.config.ts`, and can be versioned/published independently.

## Deprecation

When a package or version should no longer be used:

```bash
# Deprecate a specific version
npm deprecate @scope/package-name@1.0.0 "Use v2.0.0 instead"

# Deprecate all versions
npm deprecate @scope/package-name "This package has been renamed to @scope/new-name"
```

## What This Skill Does NOT Cover

- **Package scaffolding** (creating the package structure) → use `ts-package` skill
- **Writing tests** → use `ts-test` or `react-test` skills
- **Building the package** (tsup configuration) → use `ts-package` skill

## Checklist

When publishing a package:

1. **Verify `package.json` metadata** — name, version, description, license, exports, files, peer dependencies
2. **Verify `prepublishOnly` script** — must run typecheck, test, and build
3. **Dry run** — `npm pack --dry-run` to verify published contents
4. **Check for sensitive files** — no `.env`, credentials, `.npmrc` with tokens, or `src/` in the tarball
5. **Bump version** — `npm version patch|minor|major` following semver
6. **Publish** — `npm publish --access public` (or without `--access` for private registries)
7. **Push tags** — `git push && git push --tags`
8. **Verify install** — `npm install @scope/package-name` in a clean project to confirm it works
