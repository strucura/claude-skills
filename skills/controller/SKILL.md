---
name: controller
description: Create Laravel controllers that delegate to Form Requests, Actions, and return JsonResources. Use when the user wants to add or modify controllers.
argument-hint: "[controller or resource name]"
allowed-tools: Read, Grep, Glob, Edit, Write
---

# Controller Skill

You help create Laravel controllers that follow a strict separation of concerns: Form Requests handle authorization and validation, Actions handle business logic, and Resources control the response shape.

## Core Principles

1. **Controllers contain only application logic** — routing input to the right Action and formatting the response. No business logic ever.
2. **Every method has a dedicated Form Request** — even index/show. Each Form Request is used exactly once.
3. **Business logic lives in Actions** — controllers call `Action::make()->handle()` for write operations.
4. **Every response uses a JsonResource** — use `::make()` for singular, `::collection()` for lists. Never pass raw models.
5. **Read endpoints (index, show, edit, create) do not use Actions** — index queries directly, show/edit use route model binding, create renders a form.

## Discovery

Before creating a Controller, scan the project to understand conventions:

1. **Find existing Controllers** — `Glob` for `**/Controllers/*.php` to discover directory structure.
2. **Find existing routes** — `Grep` for `Route::` in route files to see routing patterns.
3. **Find related Form Requests, Actions, Resources** — check if they already exist for the model.

## Full CRUD Controller Template

```php
<?php

namespace App\Http\Controllers\{Domain};

use App\Http\Controllers\Controller;
use App\Domains\{Domain}\{SubDomain}\Actions\CreateAssetAction;
use App\Domains\{Domain}\{SubDomain}\Actions\DeleteAssetAction;
use App\Domains\{Domain}\{SubDomain}\Actions\UpdateAssetAction;
use App\Domains\{Domain}\{SubDomain}\Data\AssetData;
use App\Domains\{Domain}\{SubDomain}\Requests\CreateAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Requests\DestroyAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Requests\EditAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Requests\IndexAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Requests\ShowAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Requests\StoreAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Requests\UpdateAssetRequest;
use App\Domains\{Domain}\{SubDomain}\Resources\AssetResource;
use App\Models\Asset;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;
use Inertia\Response;

class AssetController extends Controller
{
    public function index(IndexAssetRequest $request): Response
    {
        $assets = Asset::query()
            ->with(['category', 'assignedUser'])
            ->paginate();

        return Inertia::render('assets/index', [
            'assets' => AssetResource::collection($assets),
            'can' => [
                'create' => Gate::allows('assets:create'),
            ],
        ]);
    }

    public function show(ShowAssetRequest $request, Asset $asset): Response
    {
        return Inertia::render('assets/show', [
            'asset' => AssetResource::make($asset->load(['category', 'assignedUser'])),
            'can' => [
                'edit' => Gate::allows('assets:edit'),
                'destroy' => Gate::allows('assets:destroy'),
            ],
        ]);
    }

    public function create(CreateAssetRequest $request): Response
    {
        return Inertia::render('assets/create');
    }

    public function store(StoreAssetRequest $request): RedirectResponse
    {
        $asset = CreateAssetAction::make()->handle(
            AssetData::fromStoreAssetRequest($request),
        );

        return redirect()->route('assets.show', $asset);
    }

    public function edit(EditAssetRequest $request, Asset $asset): Response
    {
        return Inertia::render('assets/edit', [
            'asset' => AssetResource::make($asset),
        ]);
    }

    public function update(UpdateAssetRequest $request, Asset $asset): RedirectResponse
    {
        UpdateAssetAction::make()->handle(
            $asset,
            AssetData::fromUpdateAssetRequest($request),
        );

        return redirect()->route('assets.show', $asset);
    }

    public function destroy(DestroyAssetRequest $request, Asset $asset): RedirectResponse
    {
        DeleteAssetAction::make()->handle($asset);

        return redirect()->route('assets.index');
    }
}
```

## API Controllers

For endpoints that return JSON instead of Inertia pages, return Resources directly:

```php
public function show(ShowAssetRequest $request, Asset $asset): AssetResource
{
    return AssetResource::make($asset->load(['category']));
}

public function index(IndexAssetRequest $request): \Illuminate\Http\Resources\Json\AnonymousResourceCollection
{
    $assets = Asset::query()
        ->with(['category'])
        ->paginate();

    return AssetResource::collection($assets);
}
```

## Naming Conventions

| Thing | Pattern | Example |
|---|---|---|
| Controller | `{Model}Controller` | `AssetController` |
| Route name | `{resource}.{action}` | `assets.index`, `assets.store` |

## Checklist

When creating a Controller:

1. **Ensure corresponding Form Requests, Actions, and Resources exist** (see form-request, action, resource skills)
2. **Delegate write operations to Actions** — store, update, destroy call `Action::make()->handle()`
3. **Read methods use route model binding or direct queries** — no Actions for index, show, create, edit
4. **Return all data through Resources** — `::make()` for singular, `::collection()` for lists, never raw models
5. **Use Data objects for store/update** — map from the Form Request via a static `from*` method
6. **Pass a `can` array** with only the Gate permissions the page needs
7. **Keep methods short** — most should be 3-8 lines of actual logic
