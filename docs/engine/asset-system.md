# Asset System

The runtime **AssetSystem** owns CPU-side asset records and the dependency batches used to group asset requests for larger loading operations.

## Public boundary

The Engine accesses the module through `SAssetSysAPI`. The current public surface covers lifecycle, dependency batches, model requests, model state, and borrowed model views.

The AssetSystem imports only shared paths and logging; it does not own or depend on RenderSystemVK.

## Model requests

A model request is identified by an `AssetHandle<SModelAsset>`. Requests are deduplicated by asset path, so requesting an already-known model resolves to the existing record/handle instead of creating duplicate CPU asset state.

```text
asset path
    │
    ▼
RequestModel(path, batch)
    │
    ▼
AssetHandle<SModelAsset>
    │
    ├─ GetModelState
    └─ GetModel -> borrowed SModelAssetView
```

The AssetSystem retains ownership of resolved model memory. A returned model view is read-only borrowed data, not a transfer of ownership.

## Dependency batches

A dependency batch groups handles needed by a larger loading operation such as a map transition. `LoadDependencyBatch()` currently performs the batch's CPU loading synchronously.

Batch progress reports aggregate total/completed/failed dependency counts. The batch can also expose its unique model handles so the Engine can pass the CPU-ready set to RenderSystemVK for GPU preparation.

Destroying a dependency batch destroys **batch bookkeeping**, not the asset records referenced by that batch.

## CPU/GPU separation

AssetSystem ends at CPU-ready content:

```text
VMDL file
   │
   ▼
AssetSystem / CVMDLReader
   │
   ▼
SModelAsset (CPU)
   │ AssetHandle
   ▼
RenderSystemVK::PrepareModelResources
   │
   ▼
backend-specific GPU state
```

This prevents Vulkan allocation/lifetime concerns from becoming part of the runtime asset loader.

## Fallback behavior

The AssetSystem owns a required fallback/error model. An ordinary failed model request can resolve through that fallback when it is available. Failure of the fallback itself cannot be handled in the same way because there is no valid substitute beneath it.

## Current scope

The implementation is currently strongly model-oriented. Additional asset classes can adopt the same typed-handle/record ownership model as their runtime pipelines become concrete.

!!! warning "Lifetime rule"
    Handles are references to AssetSystem records. Resolved views borrow record-owned memory and must not be retained beyond the lifetime guaranteed by the AssetSystem.
