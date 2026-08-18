# Render Commands & GPU Resources

The renderer separates reusable Vulkan command helpers from higher-level GPU resource ownership.

## Transfer commands

`CVulkanTransfer` centralizes transfer-oriented work such as staging/copy operations needed to move resource data into GPU-visible allocations. This prevents each model or texture upload path from creating its own ad-hoc transfer setup.

## Barriers

The `commands` layer also contains Vulkan barrier helpers. Synchronization and layout transitions are Vulkan-specific implementation details and should remain close to command recording rather than leaking into gameplay or asset-format code.

## CPU assets vs GPU resources

Loaded model data is CPU-side AssetSystem state. It is not automatically a renderable Vulkan resource.

```text
AssetSystem
   │
   ├─ load VMDL through CVeilModelLoader
   ▼
CPU model record / SModelAssetView
   │ AssetHandle<EAssetType::Model>
   ▼
RenderSystemVK::PrepareModelResources
   │
   ▼
GPU model resources
   │
   ▼
RenderFrame
```

The renderer owns its GPU-facing resource manager and exposes preparation/cleanup through the renderer boundary.

## Why the split matters

AssetSystem should not need to know how Vulkan buffers are allocated, staged, synchronized, or destroyed. Likewise, the renderer should not become the owner of compiled asset loading.

The CPU asset is the portable runtime content representation; the GPU resource is backend-specific prepared state. `AssetHandle<EAssetType::Model>` remains the engine-side identity used to associate those layers.

## Render extraction

The Client's renderable components and extracted `SRenderWorld` carry model handles rather than Vulkan objects. RenderSystemVK resolves those identities against its prepared GPU resources when drawing the frame.

This preserves the boundary:

```text
Gameplay/ECS
    │ model handle
    ▼
SRenderWorld
    │
    ▼
RenderSystemVK
    │
    ▼
GPU resources
```

As additional graphics features appear, the same separation should be preserved: resource managers own prepared GPU state, rendering code consumes that state, and low-level Vulkan helpers implement API mechanics.
