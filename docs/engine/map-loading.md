# Map Loading

`CVMapLoader` coordinates loading a compiled VMAP and turning it into an active client world with its required CPU and GPU resources.

## Ownership

The loader borrows the AssetSystem, RenderSystem, and client/game APIs. It does not replace those subsystem responsibilities; instead it coordinates work that crosses all three.

## Loading stages

The current state machine is explicit:

```text
Idle
 │
 ▼
Parsing
 │
 ▼
CreatingWorld
 │
 ▼
Precaching
 │
 ▼
LoadingCPUAssets
 │
 ▼
PreparingGPUResources
 │
 ▼
ActivatingWorld
 │
 ▼
Ready
```

Any unrecoverable stage can transition to `Failed`.

## Sequence

The loader first reads and validates the compiled VMAP beneath the configured map directory. It then creates an AssetSystem dependency batch and passes that batch into client-world creation.

The client creates the world and deserializes map entities. During `PrecacheMapWorld()`, entity `Precache()` implementations request their root models into the active dependency batch.

`LoadDependencyBatch()` loads those CPU models synchronously. Model loading discovers material paths, and material loading discovers texture paths, so the batch's root model set leads to the complete transitive CPU dependency chain.

After successful CPU loading, the Engine copies the unique root model handles from the batch and passes them to `RenderSystemVK::PrepareModelResources()`. GPU model preparation recursively resolves the corresponding GPU materials and textures. Only after both CPU and GPU preparation succeed does the client spawn and activate the world.

This staged design prevents gameplay from entering an active world whose required render or asset data is still missing.

## Dependency-batch semantics

A map load owns one `AssetDependencyBatchHandle`. The batch groups root model requests for that loading operation; it is not an ownership container and does not need to list every transitive material or texture explicitly.

Destroying the batch removes its bookkeeping without destroying the AssetSystem records it referenced.

The complete batch behavior, loading states and fallback rules are documented in [Asset System](asset-system.md#dependency-batches).

## Replacement boundary

Client map requests are queued by `CEngine` and processed at a safe frame boundary. `CVMapLoader::Load()` then synchronously replaces the currently loaded map.

!!! note
    The current loader is synchronous even though its stages are explicit. The state model provides a useful boundary if loading later becomes more asynchronous or exposes richer progress reporting.
