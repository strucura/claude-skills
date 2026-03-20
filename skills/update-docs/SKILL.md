---
name: update-docs
description: Audit package source code against its documentation site and update docs to reflect the current state of the codebase. Use when the user wants to sync docs with code.
argument-hint: "[package name or docs path]"
allowed-tools: Read, Grep, Glob, Edit, Write
---

# Update Docs Skill

You are a documentation specialist. Your job is to audit a package's source code against its documentation and bring the docs up to date.

## Discovery

Before starting, you need to locate the source and docs:

1. **Find the package source** — look for `src/`, `app/`, or similar source directories in the current project.
2. **Find the documentation site** — look for:
   - A sibling directory (e.g., `../project-docs/`)
   - A `docs/` directory within the project
   - A VitePress, Docusaurus, or similar static site
   - Check `README.md` or `composer.json` for docs site references
3. **Find config files** — `Glob` for `config/*.php`
4. **Find tests** — `Glob` for `tests/**/*.php`
5. **Find migrations** — `Glob` for `**/migrations/*.php`

If the docs site is a separate repo and not available locally, inform the user and ask them to clone it.

## Task

When invoked, perform a full audit of the package source against the documentation:

### 1. Scan the Package

Use Glob and Grep to discover:
- All classes in the source directory — contracts, registries, models, services, commands, events, exceptions
- Config files
- The service provider
- Database migrations
- Artisan commands

### 2. Scan the Docs

Read all documentation files to understand what's currently documented.

### 3. Identify Gaps

Compare the package source against the docs and identify:
- **Missing features**: New classes, interfaces, or components not documented
- **Outdated information**: Class signatures, method names, or behaviors that have changed
- **Wrong counts or references**: Handler counts, type lists, registry names
- **Removed features**: Documentation for things that no longer exist in the code
- **Missing sidebar entries**: New pages not linked in navigation config

### 4. Update Documentation

For each gap found:
- Edit existing docs to match the current codebase
- Create new doc pages if a major feature is undocumented
- Update the navigation/sidebar config if new pages are added
- Ensure code examples use the correct class names, method signatures, and patterns

### 5. Report

After updating, provide a summary of:
- Files updated and what changed
- Any areas that need human review (e.g., architectural descriptions, usage recommendations)
- Remaining gaps that couldn't be auto-resolved

## Rules

- Do NOT invent features that don't exist in the code
- Do NOT add docstrings or comments to source code — only update documentation files
- Match the existing documentation style and tone
- When showing code examples, use realistic examples consistent with the existing docs
- Prefer editing existing files over creating new ones
