---
name: datagrid
description: Create and manage DataGrids for tabular data display with filtering, sorting, pagination, and actions. Use when the user wants to add a new datagrid or modify an existing one.
argument-hint: "[datagrid name or description]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash(php artisan widget:register*), Bash(php artisan migrate*)
---

# DataGrid Skill

## Architecture Overview

DataGrids provide tabular data with filtering, sorting, pagination, and inline/bulk actions.

- **Route macro:** `Route::dataGrid(DataGridClass::class)` — registers API endpoints per grid
- **API endpoints:** `{routePath}/data`, `{routePath}/schema`, `{routePath}/views`, `{routePath}/actions/inline`, `{routePath}/actions/bulk`

## Discovery

Before creating a DataGrid, scan the project:

1. **Find existing DataGrids** — `Glob` for `**/*DataGrid.php` to discover directory structure and naming patterns.
2. **Find route registrations** — `Grep` for `Route::dataGrid` to see how grids are registered.
3. **Find the base class** — `Grep` for `extends DataGrid` to confirm the import path.

## Creating a DataGrid

### Template

```php
<?php

namespace App\DataGrids;

use Strucura\Dashboards\Contracts\ShouldRegisterAsWidget;
use Illuminate\Database\Query\Builder;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;
use Strucura\DataGrid\Abstracts\DataGrid;
use Strucura\DataGrid\Columns\Number; // Also: Text, DateTime

class {Name}DataGrid extends DataGrid implements ShouldRegisterAsWidget
{
    public function getWidgetName(): string
    {
        return '{Human-readable name}';
    }

    public function getWidgetDescription(): string
    {
        return '{Short description of what this grid shows}';
    }

    public function getColumns(): Collection
    {
        return collect([
            Number::make('{table}.id', 'id')
                ->header('ID')
                ->asRowKey()
                ->withoutFiltering()
                ->pinLeft(),

            // Add columns here...
        ]);
    }

    public function getQuery(): Builder
    {
        return DB::table('{table}');
    }
}
```

## Key Conventions

- **Always use table-qualified column names:** `'users.name'` not `'name'`
- **The second argument is the alias:** must be unique within the grid
- **One column must have `->asRowKey()`:** typically the ID column
- **Column types:** `Text`, `Number`, `DateTime` (from `Strucura\DataGrid\Columns`)
- **Column modifiers:**
  - `->header('Label')` — display header text
  - `->asRowKey()` — marks as unique row identifier
  - `->hidden()` — exists but not visible by default
  - `->withoutFiltering()` — disable filtering
  - `->withoutSorting()` — disable sorting
  - `->pinLeft()` — pin column to left side

## Optional Features

### Floating Filters
Quick filter UI above the grid:
```php
use Strucura\Visualizations\FloatingFilters\DateRange;

public function getFloatingFilters(): Collection
{
    return collect([
        DateRange::make('{table}.created_at', 'Created Date'),
    ]);
}
```

### Inline Actions (per-row)
```php
use Strucura\DataGrid\Actions\Action;

public function getInlineActions(): Collection
{
    return collect([
        Action::toRoute('view', 'route.name')
            ->meta('label', 'View'),
    ]);
}
```

### Bulk Actions (multi-select)
```php
public function getBulkActions(): Collection
{
    return collect([
        Action::make('delete', function ($id) {
            $model = Model::findOrFail($id);
            abort_unless(auth()->user()->can('delete', $model), 403);
            $model->delete();
            return ['success' => true];
        })->meta('label', 'Delete'),
    ]);
}
```

### Custom Key/Route
```php
public function getDataGridKey(): string
{
    return 'grids.{custom_key}';
}

public function getRoutePath(): string
{
    return 'grids/{custom-path}';
}
```

### Computed/Subquery Columns
```php
Number::make(
    '(SELECT COUNT(*) FROM related_table WHERE related_table.parent_id = main_table.id)',
    'related_count'
)
    ->header('Related Items')
    ->withoutFiltering()
    ->withoutSorting(),
```

### Scoped Queries (user-level access control)

Add `where` clauses to `getQuery()` to scope results to the authenticated user (e.g., `->where('items.user_id', auth()->id())`).

## Registration Checklist

When creating a new DataGrid:

1. **Create the PHP class** in the appropriate directory
2. **Implement `ShouldRegisterAsWidget`** if it should appear on dashboards
3. **Register the route** in your routes file:
   ```php
   Route::middleware(['auth', 'verified'])->group(function () {
       Route::dataGrid(NewDataGrid::class);
   });
   ```
4. **Add the `use` import** at the top of the routes file
5. **Run `php artisan widget:register`** to register the widget (if applicable)
6. **Verify** with `php artisan route:list --path=grids`
