# Render API & Data Boundary

The Engine communicates with the renderer through the versioned `SRenderSysAPI` table. Implementation classes remain inside the RenderSystemVK module.

## Imports

`SRenderSysImport` supplies:

- the launcher-owned SDL window as an opaque borrowed handle;
- shared path and logging services;
- callbacks for reading model, material, and texture state;
- callbacks that provide borrowed `SModelView`, `SMaterialView`, and `STextureView` values.

These callbacks are the renderer's CPU-asset access boundary. RenderSystemVK does not own AssetSystem records or retain ownership of the spans exposed through an asset view.

## Exported operations

| Operation | Responsibility |
| --- | --- |
| `Initialize` | Validate imports and create the complete renderer |
| `Shutdown` | Wait for outstanding work and release renderer-owned state |
| `PrepareModelResources` | Eagerly prepare unique CPU-ready model handles |
| `ClearModelResources` | Idle the device and invalidate cached model-related GPU resources |
| `RenderFrame` | Render one extracted view/world snapshot |
| `OnWindowPixelSizeChanged` | Record a new physical-pixel extent for deferred recreation |
| `SetVSync` | Change presentation policy and schedule safe recreation |

## Frame snapshots

`RenderFrame()` receives two client-owned, backend-neutral snapshots:

- `SRenderView` contains camera and lighting state;
- `SRenderWorld` contains model instances and optional sky state.

Model instances carry `AssetHandle<EAssetType::Model>`, transforms, and render flags. They never carry `VkBuffer`, descriptor sets, or renderer-private GPU pointers.

## Resource preparation policy

Model resources currently have two preparation paths:

1. `PrepareModelResources()` is the eager path used before activating a loaded map.
2. `PrepareWorldResources()` is a defensive frame-time path that ensures every model and standalone sky material referenced by the extracted world is available.

Both paths use `CGPUResourceManager`, whose CPU-asset-ID caches avoid recreating already prepared resources.

Preparing a model resolves its dependencies transitively:

```text
model handle
  ├─ SModelView geometry
  └─ material handles
       └─ SMaterialView texture bindings
            └─ STextureView image data
```

## Lifetime rules

- Asset views are borrowed only while the associated lookup result remains valid.
- GPU handles and pointers are private to the renderer.
- `CGPUResourceManager::Clear()` and renderer shutdown invalidate all previously returned GPU handles, pointers, descriptor sets, and fallback handles.
- `ClearModelResources()` waits for the device before clearing GPU resources.
- The sky pass clears cached resource bindings before the manager is cleared.

## Boundary rule

The Engine coordinates CPU loading and GPU preparation, but neither subsystem owns the other. AssetSystem produces portable CPU content; RenderSystemVK produces backend-specific GPU state.
