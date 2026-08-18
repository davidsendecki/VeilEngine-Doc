# Asset Handles & Lifetime

The runtime AssetSystem owns asset records. Other systems identify those records through typed `AssetHandle<T>` values rather than owning the underlying asset memory.

## Typed identity

A model request returns an `AssetHandle<SModelAsset>`. The type parameter makes it explicit what kind of asset the handle is intended to address and prevents the public API from degenerating into untyped integer IDs.

## Ownership

The central rule is:

```text
AssetSystem owns record + CPU data
            │
            ├─ AssetHandle<T> ── durable identity
            │
            └─ AssetView ─────── borrowed resolved data
```

Handles remain valid according to the lifetime of their AssetSystem records. Views returned by `GetModel()` borrow AssetSystem-owned memory and must not be retained after that asset is unloaded or the AssetSystem shuts down.

## Request and resolve

The public model workflow is:

```text
RequestModel(path, batch)
        │
        ▼
AssetHandle<SModelAsset>
        │
        ├─ GetModelState(handle)
        │
        └─ GetModel(handle, view)  [when ready]
```

The request is associated with a dependency batch. Loading the batch performs the CPU-side loading required by those requests.

## Dependency batches are not ownership groups

A dependency batch groups requests for a loading operation. Destroying the batch removes the batch bookkeeping; it does not imply that all referenced asset records are immediately destroyed.

This distinction matters for map loading, where the batch is useful for gathering and preparing dependencies but runtime asset identity belongs to the AssetSystem itself.

## GPU lifetime

An AssetHandle identifies CPU-side asset state, not a Vulkan buffer or image. RenderSystemVK prepares its own backend-specific GPU resources for requested model handles.

Keeping these lifetimes separate prevents a Vulkan allocation from becoming the canonical identity of an engine asset.
