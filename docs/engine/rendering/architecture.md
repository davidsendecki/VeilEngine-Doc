# Architecture & Ownership

RenderSystemVK is layered so high-level rendering decisions do not become mixed with raw Vulkan object management. The most important distinction is not merely which directory contains a class, but which object owns a resource and which objects only borrow it.

## Source layout

```text
src/rendersystemvk/src/
├── commands/       command recording and synchronous transfer
├── descriptors/    descriptor-set and write helpers
├── frame/          frame contexts and transient render queue
├── gpuresources/   GPU asset representations and caches
├── pass/           sky and forward rendering passes
├── pipeline/       vertex layout, pipeline descriptions and registry
├── presentation/   swapchain and presentation-resource bundle
├── render/         renderer entry point and exported API bridge
├── resources/      typed buffers, meshes and textures
├── shader/         shader registration and module management
└── vulkan/         foundational Vulkan owners
```

The `CVk*` classes under `vulkan/` are the canonical low-level wrappers. Higher layers give those objects rendering meaning and coordinate their use.

## Ownership summary

| Owner | Owned state | Important borrowed dependencies |
| --- | --- | --- |
| `CRenderSystemVK` | Every major renderer subsystem and the transient render queue | Engine imports and CPU asset views |
| `CVkContext` | Vulkan instance, surface, physical/logical device and graphics queue state | SDL window during initialization |
| `CVkAllocator` | VMA allocator | Vulkan context |
| `CVkTransfer` | Synchronous transfer command infrastructure | Context and allocator |
| `CVkPresentationResources` | Swapchain, views, depth image, per-image render-finished semaphores and tracked layouts | Context and allocator |
| `CVkFrameContext` | Command pool, primary command buffer, image-available semaphore and render fence | Vulkan context |
| `CGPUResourceManager` | GPU meshes, materials, textures, shared material descriptors, sampler and fallbacks | Allocator and transfer service |
| `CShaderManager` | Registered shader records and shader modules | Vulkan device |
| `PipelineManager` | Managed graphics pipelines | Vulkan device and shader manager |
| `ForwardPass` | Draw descriptors and per-frame draw uniform buffers | GPU resources, allocator and pipeline manager |
| `CSkyPass` | Sky geometry, frame buffers, descriptors, sampler and fallback textures | GPU resources, allocator, transfer and pipeline manager |

Borrowed dependencies must outlive their consumers. GPU work must no longer reference an owned Vulkan resource before that resource is destroyed.

## Layer model

```text
SRenderView / SRenderWorld / AssetHandle
                    │
                    ▼
CRenderSystemVK and render passes
                    │
                    ▼
GPU resource, shader and pipeline managers
                    │
                    ▼
Typed resources and command/descriptor helpers
                    │
                    ▼
CVk foundational wrappers
                    │
                    ▼
Vulkan + VMA
```

A `CVkBuffer` solves allocation and Vulkan-buffer lifetime. A `CVkVertexBuffer` gives a buffer a typed rendering role. A `CGPUStaticMesh` combines typed buffers and material handles into the renderer representation of a model. `CGPUResourceManager` owns and caches those model-level objects. A pass then consumes them while recording a frame.

## Initialization order

`CRenderSystemVK::Initialize()` creates dependencies from the bottom upward:

1. Vulkan context;
2. VMA allocator;
3. synchronous transfer service;
4. presentation-resource coordinator;
5. frame-in-flight contexts;
6. GPU-resource manager and fallbacks;
7. shader manager and registered shader modules;
8. pipeline manager;
9. rendering passes;
10. initial presentation resources and compatible pipelines when the window has a usable extent.

## Shutdown order

Shutdown waits for the device and destroys objects in reverse dependency order: queue contents, passes, pipelines, shaders, GPU resources, frame contexts, presentation resources, transfer service, allocator, and finally the Vulkan context.

Partial initialization may call `Shutdown()`, so each owned subsystem must tolerate explicit shutdown from an incomplete state.

## Extension rule

Add a rendering feature at the highest layer that can own its policy correctly. A pass may use typed renderer resources, descriptor helpers, and pipeline descriptions. Gameplay code should submit data through shared contracts rather than constructing Vulkan state. If every new feature must manually coordinate raw pools, fences, pipelines, and layouts outside RenderSystemVK, the renderer boundary has become too thin.
