# Rendering

Veil's runtime rendering backend is implemented by **RenderSystemVK** and targets Vulkan. The renderer consumes engine-defined asset handles and client-extracted frame snapshots while keeping Vulkan objects, synchronization, descriptors, and GPU-resource lifetime private to the renderer module.

`CRenderSystemVK` is the root coordinator. It owns the foundational Vulkan services, presentation resources, frame-in-flight contexts, GPU-resource cache, shader and pipeline managers, rendering passes, and the transient render queue.

## External boundary

```text
Gameplay world and components
          │ render extraction
          ▼
SRenderView + SRenderWorld
          │
          ▼
Engine / SRenderSysAPI
          │
          ▼
CRenderSystemVK
          │
          ▼
Vulkan + VMA
```

The renderer does not traverse `CWorld`, entity objects, or component storage. The Client produces backend-neutral snapshots, and the Engine submits those snapshots through `SRenderSysAPI`.

## Current runtime renderer

The current implementation provides:

- static-mesh forward rendering;
- metallic-roughness PBR materials;
- panorama and dynamic sky rendering;
- two fixed vertex texture-coordinate slots, with model metadata recording how many source channels are valid;
- two independently writable frame-in-flight contexts;
- dynamic rendering with a swapchain color attachment and shared depth attachment;
- deferred swapchain recreation for window-size and VSync changes;
- cached GPU meshes, materials, and textures with renderer-owned fallbacks;
- explicit C++/Slang descriptor binding contracts.

## CPU and GPU identities

An `AssetHandle<EAssetType::Model>` identifies CPU-side AssetSystem state. It is not a Vulkan buffer and does not directly identify a renderer-owned mesh.

During GPU preparation, `CGPUResourceManager` creates or retrieves a private `SGPUStaticMeshHandle`. Model material handles are resolved to renderer-private material handles, and material texture handles are resolved to renderer-private texture handles.

```text
AssetHandle<Model>
       │ borrowed SModelView
       ▼
SGPUStaticMeshHandle
       │
       ▼
CGPUStaticMesh → GPUMaterial → CGPUTexture
```

See [GPU Resource Management](gpu-resources.md) for the ownership and invalidation rules.

## Architectural rule

Gameplay, AssetSystem, and unrelated engine modules should not manipulate Vulkan descriptor pools, command pools, fences, image layouts, pipelines, or renderer-private GPU handles. New rendering features should enter through renderer-owned passes, managers, and shared renderer contracts.

!!! warning "Still evolving"
    The boundaries documented here describe the current implementation, not a frozen public graphics SDK. Skeletal rendering, shadows, additional light types, bindless resources, and runtime material instances are not part of the completed runtime renderer yet.
