---
name: form-request
description: Create Form Requests with validation, authorization, and Spatie permission registration. Use when the user wants to add validation, permissions, or authorization to controllers.
argument-hint: "[controller or resource name]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash(php artisan migrate*), Bash(php artisan route:list*)
---

# Form Request Skill

You help create Form Requests with Gate-based authorization backed by Spatie permissions.

## Core Principles

1. **Every controller endpoint MUST use a Form Request** — even endpoints with no validation rules need one for authorization.
2. **Every Form Request has its own dedicated Spatie permission** — permissions use the format `{resource}:{action}`.
3. **Every Form Request is used exactly once** — never reuse a Form Request across multiple controller methods.
4. **Authorization uses `Gate::allows()`** — not policies. Each Form Request checks a single permission string.

## Discovery

Before creating Form Requests, scan the project to understand conventions:

1. **Find existing Form Requests** — `Glob` for `**/Requests/*.php` to discover directory structure and naming patterns.
2. **Find existing permission registrations** — `Grep` for `Permission::create` or permission seed migrations.
3. **Find existing validation rule traits** — `Glob` for `**/*ValidationRules.php`.

## Creating a Form Request

### Template

```php
<?php

namespace App\Domains\{Domain}\{SubDomain}\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Facades\Gate;

class {Action}{Model}Request extends FormRequest
{
    public function authorize(): bool
    {
        return Gate::allows('{resource}:{action}');
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string', 'max:1000'],
        ];
    }
}
```

For endpoints with no input (index, show, destroy), return an empty `rules()` array.

## Permission Naming Convention

Permissions follow the pattern `{resource}:{action}` where resource is the plural lowercase model name. Follow `{resource}:{action}` for all controller methods.

| Controller Method | Form Request | Permission |
|---|---|---|
| `index` | `Index{Model}Request` | `{resource}:index` |
| `store` | `Store{Model}Request` | `{resource}:store` |
| `destroy` | `Destroy{Model}Request` | `{resource}:destroy` |

For non-CRUD actions, use descriptive names:
- `assets:export`, `assets:import`, `assets:archive`

## Registering Permissions

New permissions MUST be registered. Find the existing permission seed migration in the project and add the new permissions:

```php
$permissions = [
    // ... existing permissions ...
    'assets:index',
    'assets:show',
    'assets:create',
    'assets:store',
    'assets:edit',
    'assets:update',
    'assets:destroy',
];
```

Also assign new permissions to the appropriate roles:
- **owner** — gets all permissions
- **admin** — gets most permissions (check existing pattern for exclusions)
- **member** — gets read-only permissions (index, show, view-type actions)

## Usage

Type-hint the Form Request in the controller method signature. See the `controller` skill for full wiring patterns including `can` arrays for frontend permission checks.

## Validation Rules

### Use Laravel's Built-in Rules

```php
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

public function rules(): array
{
    return [
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'email', 'max:255', Rule::unique('users')->ignore($this->user()->id)],
        'status' => ['required', Rule::in(['draft', 'active', 'archived'])],
        'amount' => ['required', 'numeric', 'min:0'],
        'items' => ['required', 'array', 'min:1'],
        'items.*.name' => ['required', 'string'],
    ];
}
```

For shared rules, use a trait with methods returning rule arrays. For complex validation, implement `ValidationRule`. For conditional rules, use `Rule::requiredIf()`. For custom error messages, override `messages()`.

### Data Transformation

```php
protected function prepareForValidation(): void
{
    $this->merge([
        'slug' => Str::slug($this->input('name')),
    ]);
}
```

## Checklist

When creating Form Requests for a feature:

1. **Register all new permissions** in the Spatie seed migration
2. **Assign permissions to roles** (owner, admin, member)
3. **Pass `can` array to Inertia** for frontend permission-based UI
6. **Add validation rules** where the endpoint accepts user input
7. **Use validation rule traits** for rules shared across Form Requests
