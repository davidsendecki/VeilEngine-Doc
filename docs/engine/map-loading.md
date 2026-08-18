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

The loader first reads and validates the compiled VMAP beneath the configured map directory. The client creates the new world and map entities, then entity `Precache` calls populate an AssetSystem dependency batch.

The AssetSystem loads the CPU-side asset data. The renderer is then given the opportunity to prepare the GPU resources required by those assets. Only after both preparation phases succeed does the client spawn and activate the world.

This staged design prevents gameplay from entering an active world whose required render or asset data is still missing.

## Dependency batch

A map load owns an `AssetDependencyBatchHandle`. Entity precache requests attach required assets to this batch, allowing the engine to treat the map's resource requirements as one loading operation.

## Replacement boundary

Client map requests are queued by `CEngine` and processed at a safe frame boundary. `CVMapLoader::Load()` then synchronously replaces the currently loaded map.

!!! note
    The current loader is synchronous even though its stages are explicit. The state model provides a useful boundary if loading later becomes more asynchronous or exposes richer progress reporting.
