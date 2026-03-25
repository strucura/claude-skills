---
name: action
description: Create Action classes with optional Data objects for encapsulating business logic. Use when the user wants to add, modify, or test action classes.
argument-hint: "[action name or domain context]"
allowed-tools: Read, Grep, Glob, Edit, Write
---

# Action Skill

You help create Action classes and their associated Data objects. Actions encapsulate a single unit of business logic and are the primary place where domain work happens — controllers stay thin by delegating to actions.

## Core Principles

1. **One action, one job** — each Action class does exactly one thing.
2. **Actions use `CanMakeOrFake`** — every Action uses this trait for container resolution and testability.
3. **Simple parameters OR a Data object** — never both. Use standard parameters when there are few (≤3). Use a Data object when there are many or the data needs to be mapped from multiple sources.
4. **Data objects own their mapping** — a Data object knows how to construct itself from various sources via static `from*` factory methods.

## Discovery

Before creating an Action, scan the project to understand the conventions in use:

1. **Find existing Actions** — `Glob` for `**/Actions/*.php` to discover naming patterns and directory structure.
2. **Find the CanMakeOrFake trait** — `Grep` for `trait CanMakeOrFake` to locate it. If it doesn't exist, create it.
3. **Find existing Data objects** — `Glob` for `**/Data/*Data.php` to discover the directory pattern.

## The CanMakeOrFake Trait

This trait MUST be used on every Action class. If the trait doesn't exist in the project, create it at the conventional location (typically `app/Concerns/CanMakeOrFake.php` or similar):

```php
<?php

namespace App\Concerns;

use Closure;
use Mockery;
use Mockery\MockInterface;

trait CanMakeOrFake
{
    public static function make(): static
    {
        return app(static::class);
    }

    public static function fake(?Closure $callback = null): MockInterface
    {
        $mock = Mockery::mock(static::class);

        if ($callback) {
            $callback($mock);
        }

        app()->instance(static::class, $mock);

        return $mock;
    }
}
```

## Creating an Action — Simple Parameters

Use standard parameters on `handle()` when there are **3 or fewer** simple arguments.

### Template

```php
<?php

namespace App\Domains\{Domain}\{SubDomain}\Actions;

use App\Concerns\CanMakeOrFake;

class {ActionName}
{
    use CanMakeOrFake;

    public function handle(string $name, string $email): User
    {
        return User::create([
            'name' => $name,
            'email' => $email,
        ]);
    }
}
```

### Calling from a Controller

```php
public function store(StoreAssetRequest $request): RedirectResponse
{
    $asset = CreateAssetAction::make()->handle(
        name: $request->validated('name'),
        description: $request->validated('description'),
    );

    return redirect()->route('assets.show', $asset);
}
```

## Creating an Action — With Data Object

Use a Data object when there are **more than 3 parameters**, or when the same data needs to be constructed from multiple sources (requests, events, jobs, etc.).

### Action Template

```php
<?php

namespace App\Domains\{Domain}\{SubDomain}\Actions;

use App\Concerns\CanMakeOrFake;
use App\Domains\{Domain}\{SubDomain}\Data\{Name}Data;

class {ActionName}
{
    use CanMakeOrFake;

    public function handle({Name}Data $data): Asset
    {
        return Asset::create([
            'name' => $data->name,
            'description' => $data->description,
            'status' => $data->status,
            'assigned_to' => $data->assignedTo,
            'purchased_at' => $data->purchasedAt,
        ]);
    }
}
```

### Data Object Template

Data objects are plain PHP classes with `readonly` promoted constructor properties. They own the mapping logic from external sources via static `from*` factory methods.

```php
<?php

namespace App\Domains\{Domain}\{SubDomain}\Data;

use App\Domains\{Domain}\{SubDomain}\Requests\Store{Model}Request;
use Carbon\CarbonImmutable;

class {Name}Data
{
    public function __construct(
        public readonly string $name,
        public readonly ?string $description,
        public readonly string $status,
        public readonly ?int $assignedTo,
        public readonly ?CarbonImmutable $purchasedAt,
    ) {}

    public static function fromStoreAssetRequest(StoreAssetRequest $request): self
    {
        return new self(
            name: $request->validated('name'),
            description: $request->validated('description'),
            status: $request->validated('status', 'draft'),
            assignedTo: $request->validated('assigned_to'),
            purchasedAt: $request->validated('purchased_at')
                ? CarbonImmutable::parse($request->validated('purchased_at'))
                : null,
        );
    }

    // Add additional `from*` methods following the same pattern.
}
```

See the `controller` skill for full controller wiring patterns.

## Data Object Factory Methods

Each `from*` method maps data from a specific source. Name them after the source class:

| Source | Method Name |
|---|---|
| Store request | `fromStoreAssetRequest(StoreAssetRequest $request)` |
| Update request | `fromUpdateAssetRequest(UpdateAssetRequest $request)` |
| Array (e.g. from a job or event) | `fromArray(array $data)` |
| Another model | `fromModel(Asset $asset)` |

Type-hint the specific request class — not the generic `FormRequest` or `Request`. This makes the mapping explicit and self-documenting.

## Actions with Dependencies

Actions resolved via `::make()` support constructor injection:

```php
class CreateAssetAction
{
    use CanMakeOrFake;

    public function __construct(
        private readonly AssetNumberGenerator $generator,
    ) {}
}
```

## Testing Actions

### Testing an Action Directly

Test the action's `handle()` method by constructing a Data object (or passing parameters) directly:

```php
it('creates an asset', function () {
    $data = new AssetData(
        name: 'Laptop',
        description: 'Work laptop',
        status: 'active',
        assignedTo: null,
        purchasedAt: null,
    );

    $asset = CreateAssetAction::make()->handle($data);

    expect($asset)->name->toBe('Laptop')->status->toBe('active');
});
```

### Faking an Action — Asserting It Was Called

Use `::fake()` to replace the action with a mock when testing code that *calls* the action (e.g., a controller test). This isolates the test from the action's implementation.

```php
use App\Domains\Application\Assets\Actions\CreateAssetAction;

it('calls the create asset action', function () {
    $mock = CreateAssetAction::fake(function ($mock) {
        $mock->shouldReceive('handle')
            ->once()
            ->andReturn(Asset::factory()->make());
    });

    $this->actingAs(User::factory()->create())
        ->post(route('assets.store'), [
            'name' => 'Laptop',
            'description' => 'Work laptop',
            'status' => 'active',
        ])
        ->assertRedirect();

    // The mock assertion (->once()) verifies the action was called
});
```

Other `::fake()` patterns:
- **Ignoring** — `$mock->shouldReceive('handle')->never()` to assert the action was NOT called (e.g., validation failure tests)
- **Returning specific data** — `$mock->shouldReceive('handle')->once()->andReturn($data)` to control downstream behavior

### Testing a Data Object's Factory Methods

```php
it('creates data from a store request', function () {
    $request = StoreAssetRequest::create('/assets', 'POST', [
        'name' => 'Laptop',
        'description' => 'Work laptop',
        'status' => 'active',
        'assigned_to' => 42,
    ]);

    $data = AssetData::fromStoreAssetRequest($request);

    expect($data)->name->toBe('Laptop')->assignedTo->toBe(42);
});
```

## When to Use Simple Parameters vs Data Objects

| Scenario | Approach |
|---|---|
| ≤3 simple args | Simple parameters on `handle()` |
| >3 or multi-source | Data object with `from*` methods |

## Naming Conventions

| Thing | Pattern | Example |
|---|---|---|
| Action class | `{Verb}{Model}Action` | `CreateAssetAction`, `UpdateAssetAction`, `DeleteAssetAction` |
| Data object | `{Model}Data` | `AssetData` |
| Factory method | `from{SourceClass}` | `fromStoreAssetRequest`, `fromArray` |

## Checklist

When creating an Action:

1. **Create the Action class** with `use CanMakeOrFake`
2. **Decide: simple parameters or Data object** (≤3 params → simple, >3 → Data object)
3. **If Data object:** create it with `readonly` constructor properties and `from*` factory methods
4. **Call via `::make()->handle()`** — never `new` the Action in application code
5. **Write tests** — test the action directly AND fake it in controller/integration tests
6. **Ensure `CanMakeOrFake` trait exists** in the project
