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

For multiple entry points, add keys to `exports` following the same `import`/`require` pattern as `"."` above, and add corresponding entries to tsup's `entry` array.

## Peer Dependencies

Use `peerDependencies` for frameworks the consumer provides (React, Vue), `dependencies` for libraries bundled at runtime, and `devDependencies` for build/test tools. Use `||` ranges for multiple major versions. Use `peerDependenciesMeta.optional` for deps that enhance but aren't required.

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

Use `npm version patch|minor|major` to bump. Use `npm version 1.0.0-beta.1` for specific versions. Use `--no-git-tag-version` to skip the git tag. `npm version` updates `package.json`, creates a git commit, and creates a git tag by default.

## Registry Configuration

### Public npm Registry (Default)

No configuration needed. Publish with:

```bash
npm publish --access public
```

The `--access public` flag is required for scoped packages on their first publish (scoped packages default to private on npm).

### Private / GitHub Packages Registry

Create `.npmrc` in the package root:

```
@scope:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}
```

Replace the registry URL for self-hosted registries. For GitHub Packages, the scope must match the GitHub org/user name. **Never hardcode tokens in `.npmrc`** — use environment variables. For CI, set `NPM_TOKEN` as a secret.

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

Clean working directory → `npm run typecheck && npm run test && npm run build` → `npm pack --dry-run` → `npm version patch|minor|major` → `npm publish --access public` → `git push && git push --tags`.

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

Use `npm dist-tag add` to manage tags (`latest`, `beta`, `next`). Use `npm dist-tag ls` to list them.

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

## Checklist

1. **Verify `package.json`** — name, version, exports, files, peer dependencies, `prepublishOnly` script
2. **Dry run** — `npm pack --dry-run` to verify no sensitive files (`.env`, credentials, `src/`) are included
3. **Bump, publish, push** — `npm version` → `npm publish --access public` → `git push && git push --tags`
4. **Verify install** — `npm install @scope/package-name` in a clean project
