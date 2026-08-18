# Render Commands & GPU Resources

The renderer separates reusable Vulkan command helpers from higher-level GPU resource ownership.

## Transfer commands

`CVulkanTransfer` centralizes transfer-oriented work such as staging/copy operations needed to move resource data into GPU-visible allocations. This prevents each model or texture upload path from creating its own ad-hoc transfer setup.

## Barriers

The `commands` layer also contains Vulkan barrier helpers. Synchronization and layout transitions are Vulkan-specific implementation details and should remain close to command recording rather than leaking into gameplay or asset-format code.

## CPU assets vs GPU resources

A loaded `SModelAsset` is CPU-side asset data. It is not automatically a renderable Vulkan resource.

The map-loading pipeline makes this distinction explicit:

```text
AssetSystem
   │
   ├─ load VMDL / CPU model data
   ▼
SModelAsset
   │
   ▼
RenderSystemVK::PrepareModelResources
   │
   ▼
GPU model resources
   │
   ▼
RenderFrame
```

`CRenderSystemVK` owns a `CVulkanResourceManager` for this GPU-facing stage and exposes `PrepareModelResources` and `ClearModelResources` through the renderer boundary.

## Why the split matters

AssetSystem should not need to know how Vulkan buffers are allocated, staged, synchronized, or destroyed. Likewise, the renderer should not become the owner of source/compiled asset loading.

The CPU asset is the portable content representation; the GPU resource is backend-specific prepared state.

## Model renderer

`CVulkanModelRenderer` is the renderer-side consumer responsible for drawing prepared model resources from the extracted `SRenderWorld` using the current `SRenderView`.

As additional graphics features appear, the same separation should be preserved: resource managers own prepared GPU state, passes/renderers consume that state, and low-level Vulkan helpers implement API mechanics.
