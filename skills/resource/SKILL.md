---
name: resource
description: Create Laravel JsonResource classes for controlling API and Inertia response shapes. Use when the user wants to add or modify resource classes.
argument-hint: "[model or resource name]"
allowed-tools: Read, Grep, Glob, Edit, Write
---

# JsonResource Skill

You help create Laravel JsonResource classes that control the shape of data leaving the application — whether returned as JSON from an API endpoint or passed as props to an Inertia page.

## Core Principles

1. **Every model exposed to the outside gets a Resource** — never pass raw Eloquent models to Inertia or return them from API endpoints.
2. **Resources are the single source of truth for response shape** — the controller decides *what* to return; the Resource decides *how* it looks.
3. **One Resource per model** — a single Resource class handles the standard representation. Use `::collection()` for lists.
4. **Keep Resources thin** — they map model attributes to output keys. No business logic, no queries, no side effects.
5. **Always use static constructors** — use `{Model}Resource::make()` for singular and `{Model}Resource::collection()` for collections. Never use `new`.

## Discovery

Before creating a Resource, scan the project to understand conventions:

1. **Find existing Resources** — `Glob` for `**/Resources/*Resource.php` to discover directory structure and patterns.
2. **Check wrapping config** — `Grep` for `withoutWrapping` to see if the project disables the default `data` wrapper.

## Creating a Resource

### Template — Standard Resource

```php
<?php

namespace App\Domains\{Domain}\{SubDomain}\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class {Model}Resource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'status' => $this->status,
            'created_at' => $this->created_at?->toAtomString(),
            'updated_at' => $this->updated_at?->toAtomString(),
        ];
    }
}
```

### Template — Resource with Relationships

```php
<?php

namespace App\Domains\Application\Assets\Resources;

use App\Domains\Settings\Users\Resources\UserResource;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class AssetResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'status' => $this->status,
            'assigned_to' => UserResource::make($this->whenLoaded('assignedUser')),
            'category' => AssetCategoryResource::make($this->whenLoaded('category')),
            'tags' => TagResource::collection($this->whenLoaded('tags')),
            'created_at' => $this->created_at?->toAtomString(),
            'updated_at' => $this->updated_at?->toAtomString(),
        ];
    }
}
```

### Template — Resource with Conditional Fields

```php
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,

        // Only include when the relationship is loaded
        'owner' => UserResource::make($this->whenLoaded('owner')),

        // Only include when explicitly requested
        'secret_key' => $this->when($request->user()?->isAdmin(), $this->secret_key),
    ];
}
```

## Using Resources

- **Singular:** `AssetResource::make($asset)` — for show pages and API detail endpoints
- **Collection:** `AssetResource::collection($assets)` — for index pages and API list endpoints
- **Paginated:** `AssetResource::collection(Asset::paginate())` — pagination metadata is preserved automatically
- Works identically for Inertia props and API JSON responses

## Custom Collection Resources

Only create a dedicated `ResourceCollection` class when you need custom meta or wrapper logic on the collection itself (e.g., summary counts). This is rare — prefer `::collection()` in most cases.

## Best Practices

**Avoid:** Never use `new`, never access relationships without `whenLoaded()`, never use `parent::toArray()`, never pass raw related models.

### Use Consistent Key Naming

Use `snake_case` keys to match Laravel/database conventions:

```php
return [
    'id' => $this->id,
    'full_name' => $this->full_name,
    'created_at' => $this->created_at?->toAtomString(),
    'assigned_to' => UserResource::make($this->whenLoaded('assignedUser')),
];
```

### Date Formatting Conventions

Use `->toAtomString()` for `_at` columns, `->format('Y-m-d')` for `_on` columns.

## Naming Conventions

Name resources `{Model}Resource`. Only create `{Model}Collection` when custom collection meta is needed.

## Checklist

When creating a Resource:

1. **Create the Resource class** with explicit field listing in `toArray()`
2. **Use `::make()` and `::collection()`** — never `new`
3. **Use `whenLoaded()`** for all relationship fields
4. **Wrap related models** in their own Resource classes via `::make()` or `::collection()`
5. **Format `_at` dates with `->toAtomString()`** and `_on` dates with `->format('Y-m-d')`
6. **No business logic** — only attribute mapping and formatting
7. **Use the Resource in the controller** — never pass raw models to Inertia or API responses
