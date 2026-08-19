# Asset System

The runtime **AssetSystem** is Veil's owner of loaded CPU-side model, material, and texture data. It assigns stable typed handles, deduplicates requests by asset path, loads transitive dependencies, exposes borrowed read-only views through the runtime API, and maintains dependency batches for operations such as map loading.

This page documents the implementation currently present on the `VeilEngine` `main` branch. The final section records possible future simplifications separately; those proposals are not descriptions of current behavior.

## Responsibilities

The AssetSystem is responsible for:

- assigning and resolving `AssetHandle<EAssetType::...>` identities;
- deduplicating asset requests by logical asset path;
- tracking CPU-side loading state;
- loading compiled VMDL models;
- parsing runtime VMAT materials;
- decoding KTX2 textures into renderer-neutral CPU data;
- discovering model-to-material and material-to-texture dependencies;
- grouping root model requests into dependency batches;
- supplying borrowed CPU asset views to other runtime modules;
- providing the required error-model fallback.

The AssetSystem is deliberately **not** responsible for:

- allocating Vulkan buffers or images;
- owning GPU resource handles;
- recording render commands;
- compiling source assets in the runtime;
- traversing gameplay entities directly;
- activating or destroying gameplay worlds;
- implementing asynchronous streaming at the current stage.

## Architectural boundary

The runtime pipeline is divided into portable CPU content and backend-specific GPU resources:

```text
Compiled files on disk
        │
        ▼
AssetSystem loaders
        │
        ▼
AssetSystem-owned CPU records
        │ borrowed views + typed asset handles
        ▼
RenderSystemVK / CGPUResourceManager
        │
        ▼
Vulkan buffers, images and material resources
```

The Engine coordinates this transition during map loading. AssetSystem does not call into RenderSystemVK, and RenderSystemVK does not take ownership of AssetSystem memory.

## Ownership summary

| Data or service | Owner | Consumer | Lifetime rule |
| --- | --- | --- | --- |
| Asset record metadata | AssetSystem storage | AssetSystem | Valid until storage is cleared or AssetSystem shuts down |
| Model vertex/index data | AssetSystem | RenderSystemVK through `SModelAssetView` | View is borrowed |
| Material parameters and texture references | AssetSystem | RenderSystemVK through `SMaterialAssetView` | View is borrowed |
| Decoded texture bytes and mip metadata | AssetSystem | RenderSystemVK through `STextureAssetView` | View is borrowed |
| Dependency-batch bookkeeping | AssetSystem | Engine/map loader | Destroying a batch does not destroy records |
| GPU meshes, materials and textures | RenderSystemVK | Rendering passes | Independent from CPU view ownership |
| World/entities/components | Client/Game | Gameplay systems and extraction | Never owned by AssetSystem |

## Public module boundary

The Engine loads the module through `SAssetSysAPI`. The current API version is `2`.

### Imports

`SAssetSysImport` contains only:

- `IPaths* pPaths` for configured runtime content directories;
- `ILogger* pLogger` for diagnostics.

The import table intentionally contains no renderer API. This keeps Vulkan ownership outside the AssetSystem.

### Exported operations

```text
Lifecycle
  Initialize
  Shutdown

Dependency batches
  CreateDependencyBatch
  DestroyDependencyBatch
  LoadDependencyBatch
  GetDependencyBatchProgress

Models
  RequestModel
  GetDependencyBatchModelCount
  CopyDependencyBatchModels
  GetModelState
  GetModel

Materials
  GetMaterialState
  GetMaterial

Textures
  GetTextureState
  GetTexture
```

`RequestMaterial()` and `RequestTexture()` are currently private AssetSystem operations. Materials are discovered from models, and textures are discovered from materials. Callers therefore request root models while the AssetSystem resolves the transitive dependency chain.

## Typed asset handles

An asset handle is a small, non-owning identifier:

```cpp
AssetHandle<EAssetType::Model>
AssetHandle<EAssetType::Material>
AssetHandle<EAssetType::Texture>
```

The template category prevents a model handle from being passed to a material or texture operation. The stored ID is a `uint64_t`; zero is invalid.

Handles do not contain:

- an asset path;
- loading state;
- ownership information;
- CPU pointers;
- Vulkan resources.

Those properties belong to the AssetSystem record or renderer-side GPU cache. See [Asset Handles & Lifetime](../assets/asset-handles.md) for the shared handle contract.

## CPU loading state

Every record inherits the common state from `TAssetRecord`:

```text
Unloaded
Queued
Loading
Ready
Failed
```

The typical transition is:

```text
FindOrCreate
    │
    ▼
Unloaded ── request ──► Queued ── load ──► Loading
                                               │
                              ┌────────────────┴───────────────┐
                              ▼                                ▼
                            Ready                            Failed
```

`Ready` means the AssetSystem can expose a valid CPU view. `Failed` means the requested record could not produce its normal CPU asset. Fallback handling depends on the asset category and is documented below.

Loading is synchronous today. `Queued` and `Loading` still make transitions explicit and prevent recursive or repeated loading from being mistaken for readiness.

## Generic storage

Each asset category has an independent `TAssetStorage`:

```cpp
TAssetStorage<EAssetType::Model, SModelRecord>       m_ModelStorage;
TAssetStorage<EAssetType::Material, SMaterialRecord> m_MaterialStorage;
TAssetStorage<EAssetType::Texture, STextureRecord>   m_TextureStorage;
```

Storage has two indexes:

- a path lookup maps the normalized logical path to its handle;
- an ordered record container maps a handle ID back to its record.

`FindOrCreate()` first checks the path lookup. A new record receives `records.size() + 1` as its category-local ID. Because each handle is category-typed, model ID `1` and texture ID `1` are distinct identities.

The record allocation remains stable while the owning storage is alive. `Clear()` removes both the path lookup and all records. Existing handles and views must not be used after that point.

## Dependency batches

A dependency batch represents the assets required by one larger loading operation. Map loading is its current primary use.

The batch is identified outside AssetSystem by `AssetDependencyBatchHandle`. Internally, `SDependencyBatch` contains:

- its own batch handle;
- an ordered vector of unique model handles;
- a model-ID set used to prevent duplicate vector entries.

### What a batch means

The batch contains **root model requests**. It does not currently store a flat list of every material and texture. Those dependencies are discovered while each model is loaded:

```text
Dependency batch
   │ root model handles
   ▼
Model loading
   │ material paths from VMDL
   ▼
Material loading
   │ texture paths from VMAT
   ▼
Texture loading
```

This distinction explains why batch enumeration exposes models to RenderSystemVK: preparing a GPU model recursively prepares the model's resolved GPU materials and textures.

### Creating and populating a batch

`CreateDependencyBatch()` assigns a non-zero batch ID and inserts an empty internal batch.

`RequestModel(path, batch)` requires a valid batch. It:

1. rejects an invalid batch;
2. rejects an empty path;
3. finds or creates the model record by path;
4. changes `Unloaded` to `Queued`;
5. inserts the model handle into the batch if it is not already present;
6. returns the stable typed model handle.

Repeated requests for the same model path reuse the same AssetSystem record and produce one batch entry.

### Loading a batch

`LoadDependencyBatch()` iterates the batch's model handles. Ready models are skipped. Other valid records enter `LoadModelRecord()`, which also resolves and loads their materials and textures synchronously.

The batch returns failure if a required record cannot be processed. After loading, `GetDependencyBatchProgress()` derives aggregate totals from the current states of the root model records.

### Enumerating completed roots

The Engine uses `GetDependencyBatchModelCount()` followed by `CopyDependencyBatchModels()` to obtain the root handle array. That array is passed to `RenderSystemVK::PrepareModelResources()`.

The two-step API avoids exposing an AssetSystem-owned `std::vector` across the dynamic-module boundary.

### Destroying a batch

`DestroyDependencyBatch()` removes only the dependency collection. It does not unload the model, material, or texture records referenced by it.

This is an important invariant:

> A dependency batch groups work; it is not an ownership container for the assets.

## Map-loading sequence

`CVMapLoader` is the cross-system coordinator. Its current synchronous sequence is:

```text
CVMapLoader                     Client/Game              AssetSystem              RenderSystemVK
     │                              │                         │                           │
     ├─ parse VMAP                  │                         │                           │
     ├─ CreateDependencyBatch ─────────────────────────────► │                           │
     ├─ CreateMapWorld(batch) ────► │                         │                           │
     ├─ PrecacheMapWorld ─────────► │                         │                           │
     │                              ├─ RequestModel(path,batch)────────────────────────► │
     ├─ LoadDependencyBatch ───────────────────────────────► │                           │
     │                              │                         ├─ model/material/texture   │
     ├─ copy root model handles ◄─────────────────────────── │                           │
     ├─ PrepareModelResources ─────────────────────────────────────────────────────────► │
     ├─ ActivateMapWorld ─────────► │                         │                           │
     ▼                              ▼                         ▼                           ▼
   Ready
```

The client receives the batch through map-world creation. Entity `Precache()` calls use the Engine-provided model-request capability, and the Engine forwards those requests to AssetSystem with the active batch.

The world is activated only after both CPU loading and GPU preparation succeed. This prevents gameplay from entering a world whose required render resources are unavailable.

## Model pipeline

### Model record

`SModelRecord` combines common record metadata with:

- `SLoadedModel Model`;
- a vector of resolved material handles;
- `bLoadedFallback`, which records whether this handle should resolve to the error model.

`SLoadedModel` owns:

- model-wide vertices;
- model-wide indices;
- mesh sections;
- material paths read from VMDL;
- calculated model bounds.

Mesh sections refer to the material-handle vector through `MaterialIndex`.

### VMDL loading

`LoadModelRecord()` resolves the record path beneath `IPaths::ModelsDir()` and invokes `CVeilModelLoader`.

The loader:

1. reads the complete file into owned bytes;
2. validates the VMDL header;
3. validates serialized section ranges;
4. reads material strings;
5. reads and validates finite vertex attributes;
6. reads and validates indices;
7. reads mesh sections and validates their index/material ranges;
8. derives axis-aligned model bounds.

The resulting `SLoadedModel` contains renderer-neutral CPU data. It retains no references to the source file.

### Resolving model materials

After geometry loading succeeds, AssetSystem iterates `SLoadedModel::MaterialPaths`.

Each material path is passed to private `RequestMaterial()`. The resulting material record is loaded immediately. A successfully loaded material handle is stored in `SModelRecord::Materials`; an invalid handle is stored when the material cannot be used.

The renderer later interprets an invalid material handle as a request for its shared fallback material.

### Model view

`GetModel()` does not transfer the vectors. It fills `SModelAssetView` with borrowed pointers and counts for:

- vertices;
- indices;
- mesh sections;
- resolved material handles;
- model bounds copied by value.

The AssetSystem record remains the owner of all pointed-to memory.

## Material pipeline

### Request validation

Material requests are internal dependencies. `RequestMaterial()` rejects:

- null or empty paths;
- rooted/absolute paths;
- paths without the `.vmat` extension;
- paths containing `..` traversal components.

Valid paths are deduplicated through material storage and queued when first encountered.

### VMAT parsing

`CMaterialLoader` parses a runtime VMAT into `SLoadedMaterial`. The parsed result contains:

- a shader name;
- standard material parameters;
- texture paths paired with their material semantics.

The current runtime model-material path supports `standardmaterial`. An unsupported shader causes the material record to fail so the renderer can substitute its fallback.

Before the material becomes ready, AssetSystem clamps base color, roughness, metallic and ambient-occlusion values to valid ranges.

### Resolving texture dependencies

For every parsed texture reference, AssetSystem determines:

- its `ETextureSemantic`;
- whether it must be interpreted as sRGB or linear data.

Base-color textures are requested as sRGB. Normal, roughness, metallic and ambient-occlusion textures are requested as linear.

The resulting `SMaterialTexture` stores the path, semantic, texture handle and fallback marker. `SMaterialRecord` owns these entries and also builds `SMaterialTextureView` entries whose pointers can be exposed through the module API.

### Material view

`GetMaterial()` succeeds only for a ready record. `SMaterialAssetView` borrows:

- the shader-name string;
- the texture-view array;

and copies the standard parameters and fallback flag by value.

The renderer uses the semantic on each texture view to place the texture in the corresponding GPU material slot.

## Texture pipeline

### Request identity

Texture records are deduplicated by asset path. The first request also assigns the required color space and semantic.

If the same path is later requested with an incompatible color space or conflicting normal-map interpretation, AssetSystem rejects that use. A texture's decoding and GPU format cannot safely change according to whichever material requested it last.

### KTX2 loading

`LoadTextureRecord()` resolves the path beneath `IPaths::MaterialsDir()` and invokes `CKTX2TextureLoader`.

The loader accepts supported non-array, non-cubemap two-dimensional KTX2 textures with explicit mip data. It:

1. validates the texture shape and mip declarations;
2. transcodes supported Basis-encoded content when required;
3. copies every mip into tightly packed RGBA8 CPU storage;
4. creates one `STextureSubresource` entry per mip;
5. reconstructs supported two-component normal-map channels when the semantic is `Normal`;
6. stores the requested color-space metadata;
7. validates the complete `STextureData` layout.

`STextureData` owns the byte allocation and mip/subresource metadata.

### Texture view

`GetTexture()` requires a ready, valid record. `STextureAssetView` borrows the byte allocation and subresource array while copying dimensions, format, mip count and color space.

The view deliberately contains no KTX objects. KTX-Software is confined to AssetSystem's loading implementation.

## CPU-to-GPU preparation

After the CPU batch completes, `RenderSystemVK::PrepareModelResources()` receives the root model handles.

For each unique handle it:

1. skips a model already present in the GPU cache;
2. verifies that the CPU model is ready;
3. obtains `SModelAssetView` from AssetSystem;
4. asks `CGPUResourceManager` to create the static mesh.

`CGPUResourceManager::CreateStaticMesh()` uploads vertex and index buffers, copies mesh-section metadata, and resolves each material handle.

GPU dependency resolution then follows the CPU relationship:

```text
AssetHandle<Model>
   │
   ▼
GPUStaticMesh
   │ GPUMaterialHandle per material slot
   ▼
GPUMaterial
   │ GPUTextureHandle per texture semantic
   ▼
GPUTexture / CVulkanImage
```

The GPU manager maintains independent caches from source asset IDs to GPU handles. CPU asset handles remain the cross-system identity; GPU handles are renderer-private indexes.

## Fallback behavior

Fallback behavior intentionally differs by category:

| Failure | Result |
| --- | --- |
| Required `error.vmdl` fails during AssetSystem initialization | AssetSystem initialization fails |
| Ordinary VMDL fails | Its record becomes ready through the shared error-model resolution path |
| Model material path is invalid or material loading fails | Model stores an invalid material handle; renderer uses fallback material |
| Material shader is unsupported | Material record fails; renderer uses fallback material |
| Texture reference is invalid or texture loading fails | Material marks that texture as fallback; renderer uses fallback texture |
| GPU texture upload fails | Renderer uses its shared checkerboard texture |
| GPU material creation fails | Renderer uses its shared fallback material |

The error model is CPU content because missing geometry needs replacement geometry. The checkerboard texture and fallback material are renderer resources because they are backend-specific GPU objects.

## Lifetime and validity rules

The following rules must hold:

1. Handle ID zero is always invalid.
2. Handles identify AssetSystem records, never Vulkan objects.
3. A view never owns the memory it exposes.
4. A view is only valid while its AssetSystem record and underlying allocations remain unchanged and alive.
5. Callers must check the corresponding state or use the `Get...()` operation before consuming a view.
6. Destroying a dependency batch does not unload its assets.
7. Clearing AssetSystem storage invalidates every handle and borrowed view from that storage.
8. Clearing renderer resources invalidates GPU handles but does not invalidate AssetSystem handles.
9. Asset paths must remain logical, content-relative paths at the AssetSystem boundary.
10. Model activation must occur only after required CPU and GPU preparation succeeds.

## Shutdown and map replacement

`CAssetSystem::Shutdown()` clears:

- all dependency batches;
- model, material and texture storage;
- the error-model handle;
- the next batch identifier.

RenderSystemVK releases its GPU resources independently. Destruction order must keep imported API pointers valid until the consuming subsystem has shut down.

During map replacement, `CVMapLoader` clears its previous map bookkeeping and asks RenderSystemVK to clear model resources at the appropriate boundary. Asset records are currently broader-lived than an individual dependency batch.

## Why the representations exist

The current pipeline contains several structures because each one serves a different boundary:

| Representation | Purpose |
| --- | --- |
| VMDL/VMAT/KTX2 declarations | Serialized or parsed file representation |
| `SLoadedModel` / `SLoadedMaterial` / `STextureData` | Owning CPU loader result |
| `SModelRecord` / `SMaterialRecord` / `STextureRecord` | Handle, path, state and resolved dependency ownership |
| `SModelAssetView` / `SMaterialAssetView` / `STextureAssetView` | Borrowed cross-module access without transferring ownership |
| `GPUStaticMesh` / `GPUMaterial` / `GPUTexture` | Backend-specific renderer state |

This separation is valid, but some internal layers duplicate data and can be simplified later without changing the ownership model.

## Diagnostics checklist

When an asset does not render correctly, inspect the pipeline in this order:

1. Confirm the entity called `RequestModel()` with the active batch.
2. Confirm the returned model handle is valid and present in the batch.
3. Check `GetModelState()` after `LoadDependencyBatch()`.
4. Check VMDL validation errors and material-path indices.
5. Check material path validation and supported shader name.
6. Check texture color-space/semantic conflicts.
7. Check KTX2 validation or transcode diagnostics.
8. Confirm `PrepareModelResources()` received the model handle.
9. Confirm GPU mesh, material and texture creation did not fall back.
10. Distinguish a CPU fallback from a renderer-side GPU fallback.

## Source map

The principal implementation locations are:

```text
src/shared/include/asset/
    AssetHandle.h
    AssetLoading.h
    ModelAsset.h
    MaterialAsset.h
    TextureAsset.h
    texture/TextureData.h

src/shared/include/api/
    AssetAPI.h

src/assetsystem/src/
    AssetSystem.h / AssetSystem.cpp
    core/AssetRecord.h
    core/AssetStorage.h
    core/AssetDependencyBatch.h
    model/ModelAssetRecord.h
    model/VeilModelLoader.h / .cpp
    material/MaterialAsset.h
    material/MaterialLoader.h / .cpp
    texture/TextureAsset.h
    texture/KTX2TextureLoader.h / .cpp

src/engine/src/map/
    VMapLoader.h / .cpp

src/rendersystemvk/src/
    render/RenderSystemVK.h / .cpp
    gpuresources/GPUResourceManager.h / .cpp
```

## Known limitations and possible future refactor

!!! note "Deferred work"
    This section records possible simplifications. It does not redefine the current implementation, and none of these changes are required before continuing work on other engine features.

Current areas that may justify a future cleanup include:

- model, material and texture data each pass through multiple internal representations;
- `SModelRecord` separates `SLoadedModel` from its resolved material handles;
- material textures are stored once as owning entries and again as view entries;
- material parameter and texture-semantic concepts currently carry model-oriented names;
- the synchronous batch has a progress API designed for a potentially richer future loader;
- the batch maintains both ordered handles and an identifier set;
- material and texture dependency loading is synchronous and nested inside model loading.

A future refactor may consolidate the final CPU representation around one owning `SModelAsset`, `SMaterialAsset`, and `STextureAsset`, with generic storage entries providing handle/path/state metadata. Borrowed API views would remain because they represent the dynamic-module ownership boundary.

Any future refactor should preserve these architectural constraints:

- keep category-typed asset handles;
- keep dependency batches as the map-loading dependency boundary;
- keep path deduplication;
- keep AssetSystem ownership of CPU data;
- keep RenderSystemVK ownership of GPU data;
- keep borrowed renderer-neutral API views;
- keep VMDL/VMAT/KTX validation inside the asset-loading boundary;
- do not introduce asynchronous loading merely as part of a structural cleanup;
- do not turn dependency batches into asset-ownership groups.

The refactor should be driven by a concrete limitation or by the addition of another asset category, not solely by a desire to generalize the system further.
