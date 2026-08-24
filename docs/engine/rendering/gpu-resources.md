# GPU Resource Management

`CGPUResourceManager` converts borrowed CPU asset views into renderer-owned Vulkan resources. It owns the resulting meshes, materials, textures, cache mappings, material descriptor infrastructure, and renderer fallbacks.

## Resource graph

```text
AssetHandle<Model>
        │
        ▼
CGPUStaticMesh
  ├─ CVkMeshBuffer
  │    ├─ CVkVertexBuffer
  │    └─ CVkIndexBuffer
  ├─ copied mesh sections
  └─ SGPUMaterialHandle[]
             │
             ▼
        GPUMaterial
          ├─ CVkUniformBuffer
          ├─ CVkDescriptorSet
          └─ SGPUTextureHandle[]
                       │
                       ▼
                  CGPUTexture
                       └─ CVkTexture2D
```

CPU asset handles remain the identity shared across modules. `SGPUStaticMeshHandle`, `SGPUMaterialHandle`, and `SGPUTextureHandle` are renderer-private identities used to resolve manager-owned objects.

## Static meshes

`CreateStaticMesh()`:

1. returns the existing cached handle when the CPU model was already prepared;
2. validates the borrowed `SModelView`;
3. resolves each material handle to a GPU material or fallback;
4. uploads the interleaved vertex data and index data into `CVkMeshBuffer`;
5. copies mesh-section metadata;
6. caches the resulting GPU handle by model asset ID.

Each `SModelVertex` contains position, normal, tangent, and two fixed UV slots. `TexCoordChannelCount` records whether one or both source channels are meaningful; no separate UV storage buffer or mesh descriptor set is required.

## Materials

`GPUMaterial` owns:

- the runtime material type;
- an uploaded material parameter uniform buffer;
- resolved texture handles;
- a descriptor set allocated from the manager-owned material pool.

The manager owns the material descriptor-set layout, the common material sampler, and a pool sized for the supported material capacity. Material descriptor sets expose the parameter buffer and the five standard texture bindings.

## Textures

`CGPUTexture` owns the GPU image represented by `CVkTexture2D`. Texture data is uploaded from a borrowed `STextureView` through the shared synchronous transfer service.

Texture cache entries are keyed by the CPU texture asset identity. The renderer therefore uploads a shared texture asset once even when multiple materials reference it.

## Resource construction context

`SGPUResourceContext` bundles non-owning dependencies needed during construction:

- logical device;
- allocator;
- transfer service;
- shared material descriptor pool and layout;
- shared material sampler.

The context is built by `CGPUResourceManager` for each construction operation. GPU resources may use it during initialization but must not store the context or its pointers.

## Fallbacks

The renderer owns:

- a checkerboard fallback texture;
- a fallback material;
- sky-specific fallback images owned by `CSkyPass`.

Invalid, unavailable, or unsupported material requests resolve to the fallback material. Invalid or failed material texture dependencies resolve to the checkerboard texture. Failed results may be cached against their source asset IDs to avoid repeating the same unsuccessful creation work.

## Capacity

The current manager reserves descriptor/resource policy for:

- up to 4096 GPU materials;
- up to 4096 cached static meshes.

`ForwardPass` independently supports up to 8192 static-mesh draw entries in one prepared frame.

These are implementation limits, not serialized asset-format limits.

## Invalidation

Pointers returned by `GetStaticMesh()`, `GetMaterial()`, and `GetTexture()` remain manager-owned. All GPU handles, returned pointers, resource descriptor sets, and fallback handles become invalid when `Clear()` or `Shutdown()` is called.

Code must not store these values across a model-resource clear. Persistent gameplay and asset state should retain CPU asset handles instead.
