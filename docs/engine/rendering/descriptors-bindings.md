# Descriptors & Shader Bindings

Veil uses an explicit descriptor contract shared between C++ and Slang. Set and binding numbers are deliberately written down in both languages rather than generated automatically.

The authoritative mirrored files are:

- C++: `src/shared/include/render/ShaderBindingContract.h`;
- Slang: `assets/shaders/core/ShaderBindingContract.slang`.

!!! danger "Manual mirror"
    Never change a descriptor set or binding number on only one side. The C++ layouts, pipeline layouts, descriptor writes, and Slang declarations must remain identical.

## StandardModel contract

### Set 0: draw resources

Set 0 is owned by the rendering pass preparing the draw.

| Binding | Resource | Current owner/use |
| ---: | --- | --- |
| 0 | `ModelDrawParameters` | `ForwardPass` dynamic per-draw uniform data |
| 6 | bone-matrix storage | Reserved by the skeletal StandardModel shader path |

### Set 1: material resources

Set 1 is owned by `GPUMaterial` and allocated from `CGPUResourceManager`'s shared material descriptor pool.

| Binding | Resource |
| ---: | --- |
| 0 | material parameters |
| 1 | base-color texture |
| 2 | normal texture |
| 3 | roughness texture |
| 4 | metallic texture |
| 5 | ambient-occlusion texture |

The C++ `TextureBindings` array is ordered to match the standard material texture-slot ordering.

## Sky contract

Sky rendering uses one descriptor set owned by `CSkyPass`.

| Set | Binding | Resource |
| ---: | ---: | --- |
| 0 | 0 | sky view parameters |
| 0 | 1 | sky material parameters |
| 0 | 7 | primary sky texture |
| 0 | 8 | cloud texture |
| 0 | 9 | star texture |

## Descriptor ownership

| Descriptor state | Owner |
| --- | --- |
| Forward draw layout, pool and per-frame sets | `ForwardPass` |
| Standard material layout, pool and material sets | `CGPUResourceManager` / `GPUMaterial` |
| Sky layout, pool and per-frame sets | `CSkyPass` |

Descriptor sets allocated from a pool become invalid when that pool is reset or destroyed. Material descriptor sets also become invalid when the GPU-resource manager is cleared.

## Descriptor helpers

`CVkDescriptorSet` wraps a borrowed descriptor-set handle and its associated layout identity. It does not own the pool from which the set was allocated.

`CVkDescriptorWriter` accumulates buffer and image writes for one destination set. Descriptor information is stored by value, and `VkWriteDescriptorSet` structures are constructed only during `Update()`. This prevents Vulkan write structures from retaining pointers invalidated by vector reallocation.

The writer supports:

- uniform buffers;
- dynamic uniform buffers;
- storage buffers;
- combined image samplers.

Each binding/array-element pair may be written only once per accumulated update. `Reset()` clears both the queued writes and any validation failure.

## Dynamic draw offsets

Each forward frame slot owns a large persistently mapped uniform buffer. Draw blocks are separated by an aligned stride derived from `minUniformBufferOffsetAlignment`.

The set-0 descriptor references the frame buffer once. During recording, `ForwardPass` supplies the matching dynamic byte offset for each draw rather than allocating one descriptor set per object.

## Extension checklist

When adding a descriptor resource:

1. assign it to the pass/resource owner that controls its lifetime;
2. add the number to both binding-contract files;
3. update the Vulkan descriptor-set layout;
4. update descriptor-pool capacity and type counts;
5. update the descriptor writer call;
6. include the layout in the correct pipeline set order;
7. update the shader declaration;
8. document the new binding in this page.
