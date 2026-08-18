# Asset Handles & Lifetime

The runtime AssetSystem owns asset records. Other systems identify those records through category-typed `AssetHandle<Type>` values rather than owning the underlying asset memory.

## Asset categories

Asset handles are now typed by the `EAssetType` enum instead of by the concrete asset-data structure:

```cpp
AssetHandle<EAssetType::Model>
AssetHandle<EAssetType::Material>
AssetHandle<EAssetType::Texture>
```

`EAssetType` currently defines `Model`, `Material`, and `Texture`. This keeps handle type-safety without requiring declarations such as `AssetHandle<SModelAsset>` merely to identify the asset category.

## Handle representation

`AssetHandle<Type>` is a lightweight, non-owning identifier containing one AssetSystem-assigned `uint64_t` ID. Zero represents an invalid handle.

The handle is intentionally trivially copyable, standard-layout, and the same size as `uint64_t`, making it suitable for engine data structures and module API boundaries.

## Ownership

The central rule is:

```text
AssetSystem owns record + CPU data
            │
            ├─ AssetHandle<EAssetType::...> ── durable identity
            │
            └─ AssetView ───────────────────── borrowed resolved data
```

Handles remain valid according to the lifetime of their AssetSystem records. Views returned by `GetModel()` borrow AssetSystem-owned memory and must not be retained after the corresponding data becomes invalid or the AssetSystem shuts down.

## Model request and resolve

The current model workflow uses:

```text
RequestModel(path, batch)
        │
        ▼
AssetHandle<EAssetType::Model>
        │
        ├─ GetModelState(handle)
        │
        └─ GetModel(handle, view)  [when ready]
```

The request is associated with a dependency batch. Loading the batch performs the CPU-side loading required by those requests.

## Client boundary

The game/client no longer needs a direct AssetSystem API pointer for model requests. `SGameImport` exposes engine-owned asset capabilities such as `RequestModel` and `GetModel` using `AssetHandle<EAssetType::Model>`.

This keeps the Client dependent on the capability supplied by the Engine rather than on AssetSystem implementation/API ownership.

## Dependency batches are not ownership groups

A dependency batch groups requests for a loading operation. Destroying the batch removes the batch bookkeeping; it does not imply that all referenced asset records are immediately destroyed.

## GPU lifetime

An asset handle identifies AssetSystem-side CPU asset state, not a Vulkan buffer or image. RenderSystemVK prepares and owns its backend-specific GPU resources separately.

Keeping these lifetimes separate prevents a Vulkan allocation from becoming the canonical identity of an engine asset.
