---
name: chart
description: Create and manage Charts for data visualization. Use when the user wants to add a new chart, modify an existing chart, or work with chart visualizations.
argument-hint: "[chart name or description]"
allowed-tools: Read, Grep, Glob, Edit, Write, Bash(php artisan widget:register*), Bash(php artisan migrate*)
---

# Chart Skill

You help create and manage Charts powered by the `strucura/charts` package.

## Architecture Overview

Charts provide data visualization with support for bar, line, area, and pie chart types.

- **Route macro:** `Route::chart(ChartClass::class)` — registers 2 API endpoints per chart
- **API endpoints:** `{routePath}/schema` (GET) and `{routePath}/data` (POST)

## Discovery

Before creating a Chart, scan the project:

1. **Find existing Charts** — `Glob` for `**/*Chart.php` to discover directory structure and naming patterns.
2. **Find route registrations** — `Grep` for `Route::chart` to see how charts are registered.
3. **Find the base class** — `Grep` for `extends Chart` to confirm the import path.

## Creating a Chart

### Template

```php
<?php

namespace App\Charts;

use Strucura\Dashboards\Contracts\ShouldRegisterAsWidget;
use Illuminate\Database\Query\Builder;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;
use Strucura\Charts\Abstracts\Chart;
use Strucura\Charts\Datasets\Bar;
use Strucura\Charts\Datasets\Line;
use Strucura\Charts\Datasets\Area;
use Strucura\Charts\Labels\Label;
use Strucura\Visualizations\FloatingFilters\DateRange;

class {Name}Chart extends Chart implements ShouldRegisterAsWidget
{
    public function getWidgetName(): string
    {
        return '{Human-readable name}';
    }

    public function getWidgetDescription(): string
    {
        return '{Short description of what this chart shows}';
    }

    public function getLabel(): Label
    {
        return Label::make("{SQL expression for grouping}", '{alias}')
            ->header('{X-axis label}');
    }

    public function getDatasets(): Collection
    {
        return collect([
            Line::make('{aggregate expression}', '{Series Label}'),
        ]);
    }

    public function getQuery(): Builder
    {
        return DB::table('{table}')
            ->groupByRaw("{same SQL expression as label}")
            ->orderByRaw("{same SQL expression as label}");
    }
}
```

## Key Conventions

- **Label** defines the x-axis grouping (typically a date format or category)
- **Datasets** define the y-axis series — use `Bar`, `Line`, or `Area`
- **The query MUST include `groupByRaw` and `orderByRaw`** matching the label expression
- **Dataset types:** `Bar::make(expr, label)`, `Line::make(expr, label)`, `Area::make(expr, label)`
- The first dataset type determines chart rendering (BarChart, AreaChart, LineChart)

## Common Label Patterns

```php
// Monthly grouping
Label::make("DATE_FORMAT(table.created_at, '%Y-%m')", 'month')->header('Month');

// Daily grouping
Label::make("DATE(table.created_at)", 'day')->header('Day');

// Category grouping
Label::make("table.category", 'category')->header('Category');
```

## Optional: Floating Filters

```php
use Strucura\Visualizations\FloatingFilters\DateRange;

public function getFloatingFilters(): Collection
{
    return collect([
        DateRange::make('{table}.created_at', 'Date Range'),
    ]);
}
```

## Custom Key/Route

```php
public function getChartKey(): string
{
    return 'charts.{custom_key}';
}

public function getRoutePath(): string
{
    return 'charts/{custom-path}';
}
```

## Chart Schema (returned by `/schema` endpoint)

```typescript
interface ChartSchema {
    chart_key: string;
    label: { field: string; header: string; meta: Record<string, unknown> };
    datasets: { field: string; header: string; type: 'bar'|'line'|'area'|'pie'; meta: Record<string, unknown> }[];
    floating_filters: DataGridFloatingFilter[];
}
```

## Registration Checklist

When creating a new chart:

1. **Create the PHP class** in the appropriate directory
2. **Implement `ShouldRegisterAsWidget`** if it should appear on dashboards
3. **Register the route** in your routes file:
   ```php
   Route::middleware(['auth', 'verified'])->group(function () {
       Route::chart(NewChart::class);
   });
   ```
4. **Add the `use` import** at the top of the routes file
5. **Run `php artisan widget:register`** to register the widget (if applicable)
6. **Verify** the chart appears in the widget picker
