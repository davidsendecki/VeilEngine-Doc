# VMDL Model Format

VMDL is Veil's compiled runtime model format. Version 1 uses a block directory so independent model-data categories can evolve without turning the file into one monolithic structure.

!!! warning "Format under development"
    VMDL is versioned but still evolving. Version `1` describes the current binary contract; it should not yet be treated as a permanent compatibility promise.

## Outer layout

```text
VMDLHeader
VMDLBlockEntry[BlockCount]
...
block payloads
```

The 16-byte `VMDLHeader` contains the `VMDL` magic, format version, block count, and expected complete file size.

Each 24-byte `VMDLBlockEntry` identifies a block type/version and its file offset/size. Readers can therefore validate and locate blocks without assuming every possible block is present.

## Block types

The current enumeration defines architectural slots for:

```text
Metadata
Strings
Scene
Meshes
Materials
Skeleton
Animations
Collision
Physics
LODs
Bounds
```

The enumeration is broader than the currently completed static-model pipeline. A named block type means the format has an assigned category for that data; it does **not** mean the corresponding authoring/runtime feature is complete.

## Static mesh representation

The current mesh block begins with `VMDLMeshBlockHeader` and serializes each mesh as:

```text
VMDLMesh
VMDLVertex[VertexCount]
uint32_t[IndexCount]
```

`VMDLMesh` contains a name string-table index, material index, vertex/index counts, and local bounds.

The current 48-byte runtime vertex contains:

```text
Position : float3
Normal   : float3
TexCoord : float2
Tangent  : float4
```

## Strings and materials

Meshes reference names/materials through indices rather than embedding repeated full strings in every mesh record. This keeps binary references compact and separates mesh geometry from string/material data.

## Runtime loading

The AssetSystem model loader is implemented by `CVeilModelLoader`. It replaced the earlier `CVMDLReader` implementation and is responsible for validating and decoding compiled VMDL data into the runtime model representation.

The loader uses the shared `CBinaryReader` for bounds-checked binary access. Binary-range validation remains part of the loading boundary before data is exposed to the AssetSystem.

```text
.vmdl
  │
  ▼
CVeilModelLoader
  │
  ▼
AssetSystem-owned model record / CPU model data
  │
  ├─ AssetHandle<EAssetType::Model>
  └─ borrowed SModelAssetView
  │
  ▼
RenderSystemVK GPU preparation
```

The loader and AssetSystem produce CPU-side runtime content. Vulkan buffers and other backend resources are not part of the VMDL ownership model.

## Evolution rule

When new blocks become concrete, document their exact serialized structures, block versions, validation requirements, and relationships to existing indices. Avoid documenting planned skeleton/animation/physics layouts before those contracts actually exist in code.
