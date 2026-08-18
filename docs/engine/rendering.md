# Rendering

Veil's runtime rendering backend is implemented by **RenderSystemVK** and targets Vulkan. The public renderer boundary consumes engine-defined asset handles and extracted render snapshots; Vulkan details remain private to the renderer module.

!!! warning "Active development"
    The rendering architecture is undergoing active development. Treat current internal class/folder boundaries as implementation state rather than a frozen graphics API.

## External boundary

The Engine interacts with the renderer through `SRenderSysAPI`. At the current level, the important operations are resource preparation/cleanup, rendering a frame, reacting to physical window-size changes, and selecting VSync policy.

```text
Client / gameplay
      │ extracts
      ▼
SRenderView + SRenderWorld
      │
      ▼
Engine
      │ SRenderSysAPI
      ▼
RenderSystemVK
      │
      ▼
Vulkan / VMA
```

The renderer does not traverse `CWorld`, entity objects, or component storage directly.

## CPU/GPU asset boundary

AssetSystem owns CPU model data. RenderSystemVK borrows the AssetSystem API during initialization and prepares backend-specific GPU resources from CPU-ready model handles before the owning map is activated.

An `AssetHandle<SModelAsset>` is therefore an engine asset identity, not a Vulkan resource handle.

## Internal layering

The renderer separates higher-level rendering behavior from Vulkan mechanics. The current implementation includes backend context/allocation, command/transfer helpers, presentation, renderer/resource management, and Vulkan object wrappers.

A useful conceptual stack is:

```text
Render snapshots / asset handles
          │
          ▼
High-level renderer and GPU resources
          │
          ▼
Rendering/presentation operations
          │
          ▼
Vulkan wrappers + synchronization helpers
          │
          ▼
Vulkan + VMA
```

## Presentation

Swapchain and presentation behavior are renderer-owned. Window changes are reported to the renderer, which defers swapchain recreation to a safe rendering boundary rather than rebuilding presentation state inside the platform callback.

## Direction

New graphics features should normally enter through renderer-owned abstractions. Gameplay and unrelated runtime modules should not manipulate descriptor pools, command pools, fences, image layouts, or raw Vulkan pipelines directly.
