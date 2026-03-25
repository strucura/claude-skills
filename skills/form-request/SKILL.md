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

### Template — Authorization Only (No Validation)

Use for index, show, create (form display), edit (form display), and destroy endpoints.

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
        return [];
    }
}
```

### Template — With Validation

Use for store and update endpoints, or any endpoint that accepts user input.

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

## Permission Naming Convention

Permissions follow the pattern `{resource}:{action}` where resource is the plural lowercase model name.

| Controller Method | Form Request | Permission |
|---|---|---|
| `index` | `Index{Model}Request` | `{resource}:index` |
| `show` | `Show{Model}Request` | `{resource}:show` |
| `create` | `Create{Model}Request` | `{resource}:create` |
| `store` | `Store{Model}Request` | `{resource}:store` |
| `edit` | `Edit{Model}Request` | `{resource}:edit` |
| `update` | `Update{Model}Request` | `{resource}:update` |
| `destroy` | `Destroy{Model}Request` | `{resource}:destroy` |

Examples for an Asset resource:
- `assets:index`, `assets:show`, `assets:create`, `assets:store`, `assets:edit`, `assets:update`, `assets:destroy`

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
        'description' => ['nullable', 'string', 'max:1000'],
        'status' => ['required', Rule::in(['draft', 'active', 'archived'])],
        'amount' => ['required', 'numeric', 'min:0'],
        'date' => ['required', 'date'],
        'items' => ['required', 'array', 'min:1'],
        'items.*.name' => ['required', 'string'],
        'items.*.quantity' => ['required', 'integer', 'min:1'],
        'related_id' => ['required', 'exists:related_table,id'],
        'password' => ['required', 'string', Password::default(), 'confirmed'],
    ];
}
```

### Reusable Validation Rule Traits

For rules shared across multiple Form Requests, create a trait:

```php
<?php

namespace App\Concerns;

trait AssetValidationRules
{
    protected function assetNameRules(): array
    {
        return ['required', 'string', 'max:255'];
    }

    protected function assetDescriptionRules(): array
    {
        return ['nullable', 'string', 'max:1000'];
    }
}
```

Then use in Form Requests:

```php
class StoreAssetRequest extends FormRequest
{
    use AssetValidationRules;

    public function authorize(): bool
    {
        return Gate::allows('assets:store');
    }

    public function rules(): array
    {
        return [
            'name' => $this->assetNameRules(),
            'description' => $this->assetDescriptionRules(),
        ];
    }
}
```

### Custom Rules

For complex validation, check for existing custom rule classes in the project. Create new ones when Laravel's built-in rules are insufficient:

```php
<?php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ValidAssetCode implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (!preg_match('/^[A-Z]{3}-\d{4}$/', $value)) {
            $fail('The :attribute must be a valid asset code (e.g., ABC-1234).');
        }
    }
}
```

### Custom Error Messages

```php
public function messages(): array
{
    return [
        'name.required' => 'Please provide a name.',
        'items.min' => 'At least one item is required.',
    ];
}
```

### Conditional Rules

```php
public function rules(): array
{
    return [
        'reason' => [Rule::requiredIf($this->input('status') === 'cancelled'), 'string'],
    ];
}
```

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

1. **Create a Form Request for EVERY controller method** — even index/show
2. **Each Form Request is used exactly once** — never reuse across methods
3. **Each Form Request has a dedicated permission** — `{resource}:{action}` format
4. **Register all new permissions** in the Spatie seed migration
5. **Assign permissions to roles** (owner, admin, member)
6. **Type-hint the Form Request** in the controller method signature
7. **Use `$request->validated()`** to access only validated data — never `$request->all()`
8. **Pass `can` array to Inertia** for frontend permission-based UI
9. **Add validation rules** where the endpoint accepts user input
10. **Use validation rule traits** for rules shared across Form Requests
