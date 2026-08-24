# Vulkan Foundation

The lower renderer layers provide explicit lifetime ownership and reusable Vulkan mechanics. They remain implementation details of RenderSystemVK rather than public engine or gameplay APIs.

## Foundational services

### `CVkContext`

Owns the Vulkan instance, SDL surface, selected physical device, logical device, graphics/presentation queue, and queue-family information needed by the renderer.

The context outlives every Vulkan object created from its instance and device.

### `CVkAllocator`

Owns the VMA allocator created from the context's instance, physical device, and logical device. It must outlive all buffers and images allocated through it but must be destroyed before the logical device.

### `CVkTransfer`

Centralizes synchronous staging and transfer work used during resource initialization. GPU resources borrow it while uploading buffer and image data instead of creating ad-hoc command pools and fences for each asset type.

## Vulkan object owners

| Wrapper | Responsibility |
| --- | --- |
| `CVkBuffer` | Generic VMA-backed Vulkan buffer allocation |
| `CVkImage` | Generic VMA-backed Vulkan image and view state |
| `CVkCommandPool` | Command-pool ownership and command-buffer allocation |
| `CVkFence` | Fence creation, wait, reset and lifetime |
| `CVkSemaphore` | Semaphore creation and lifetime |
| `CVkDescriptorPool` | Descriptor-pool ownership and allocation |
| `CVkDescriptorSetLayout` | Descriptor-set-layout ownership |
| `CVkShaderModule` | Vulkan shader-module ownership |
| `CVkGraphicsPipeline` | Graphics pipeline and pipeline-layout ownership |
| `CVkSampler` | Vulkan sampler ownership |

These wrappers encapsulate Vulkan lifetime and validation mechanics. They do not decide which scene feature needs the object.

## Typed renderer resources

The `resources/` layer builds rendering-specific types over foundational objects:

| Resource | Role |
| --- | --- |
| `CVkVertexBuffer` | Vertex-input storage |
| `CVkIndexBuffer` | Indexed-draw storage |
| `CVkUniformBuffer` | Host-visible uniform data, including mapped writes |
| `CVkStorageBuffer` | Shader storage-buffer data |
| `CVkMeshBuffer` | Paired vertex and index buffers for a mesh |
| `CVkTexture2D` | Two-dimensional sampled image and upload state |

Typed resources are still lower-level than asset-level GPU objects. `CGPUStaticMesh`, `GPUMaterial`, and `CGPUTexture` combine them with renderer metadata and handles.

## Command recording

`CVkCommandList` wraps a borrowed recording command buffer and provides focused operations such as binding pipelines, vertex/index buffers, and descriptor sets. It does not own or submit the command buffer.

`CVkFrameContext` owns the primary per-frame command buffer and submission synchronization. `CVkTransfer` owns the separate infrastructure used for synchronous resource uploads.

## Pipeline helpers

`CVkVertexLayout` owns vertex binding/attribute descriptions. `CGraphicsPipelineBuilder` owns a higher-level managed pipeline description. `PipelineManager` converts that description into an owned `CVkGraphicsPipeline` associated with a stable ID.

These classes solve separate problems and should not be collapsed merely because each eventually contributes to Vulkan pipeline creation.

## Explicit shutdown

Renderer wrappers use explicit `Initialize()`/`Shutdown()` lifetime management. Their destructors do not replace the renderer's dependency-aware shutdown order.

Before destroying or replacing a resource, callers must ensure pending GPU work can no longer reference it. Root-level operations such as renderer shutdown and model-resource clearing wait for the device at the appropriate boundary.
