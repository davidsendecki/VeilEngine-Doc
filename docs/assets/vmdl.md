# VMDL Model Format

VMDL is Veil's compiled runtime model format. Version 1 stores a validated static-model representation that can be loaded directly into renderer-neutral CPU data and then prepared by RenderSystemVK.

!!! warning "Format under development"
    VMDL is versioned, but version `1` is not yet a permanent compatibility promise. A loader rejects unsupported versions and asks for the model to be recompiled.

## File layout

The file begins with a fixed 72-byte `VMDLHeader`. It contains counts and absolute offsets for every serialized array:

```text
VMDLHeader
SModelMeshSection[MeshCount]
uint32_t material-path-offsets[MaterialCount]
SModelVertex[VertexCount]
uint32_t indices[IndexCount]
SModelCollisionBox[CollisionBoxCount]  # optional
material-path string data
```

The header records:

- `VMDL` magic and format version;
- serialized header size;
- mesh-section count/offset;
- material-path count/offset;
- vertex count/offset;
- number of valid source UV channels;
- index count/offset;
- collision-box count/offset;
- string-data size/offset;
- complete file size.

All offsets are absolute file offsets. A collision-box offset of zero is valid only when the collision-box count is zero.

## Vertex representation

`SModelVertex` is a fixed 56-byte interleaved vertex:

```text
Position     : float3
Normal       : float3
Tangent      : float4
TexCoords[0] : float2
TexCoords[1] : float2
```

`MaximumModelTexCoordChannels` is currently `2`. Both fixed UV slots are serialized and uploaded for every vertex. `VMDLHeader::TexCoordChannelCount` records whether one or both channels came from the source model.

This design keeps the renderer vertex ABI fixed. A model with one source UV channel uses UV0 and leaves UV1 at its default value. UV data is not stored in a separate flattened array or uploaded through a storage buffer.

## Mesh sections

`SModelMeshSection` identifies a draw range and its material slot:

- `MaterialIndex` selects the corresponding material-path entry;
- `FirstVertex` and `VertexCount` describe the section's vertex range;
- `FirstIndex` and `IndexCount` describe its index range.

All sections reference the shared model vertex and index arrays. Multiple sections can use different material slots while independently using the same UV channel numbers. UV coordinates belong to vertices, and texture selection belongs to the material assigned to the section.

## Material paths

The material section stores one `uint32_t` string-data offset per material slot. Paths are stored in the trailing string-data region and remain asset-relative.

At runtime, AssetSystem resolves these paths to `AssetHandle<EAssetType::Material>` values. `SModelView::MaterialSlots` and `MaterialHandles` correspond one-to-one, using the indices referenced by mesh sections.

## Collision boxes

VMDL can store model-local oriented boxes. Each `SModelCollisionBox` contains:

- center position relative to the model origin;
- normalized quaternion in X, Y, Z, W order;
- full width, height, and depth rather than half-extents.

The loader rejects non-finite values, non-positive sizes, degenerate quaternions, and quaternions outside the normalization tolerance.

Collision boxes are CPU asset data. PhysicsSystem consumes them synchronously when creating a model body; one model body receives one Box3D hull shape per authored collision box. RenderSystemVK does not upload them as rendering geometry. See [Physics Bodies & Collision](../engine/physics/bodies-collision.md#model-bodies) for the runtime construction rules.

## Runtime loading

`CVeilModelLoader` validates and decodes the file into `SModelData`:

```text
.vmdl
  │
  ▼
CVeilModelLoader
  │
  ▼
AssetSystem-owned SModelData
  │
  ├─ AssetHandle<EAssetType::Model>
  └─ borrowed SModelView
       ├─ PhysicsSystem consumes collision data
       └─ RenderSystemVK prepares GPU geometry/materials
```

The loader validates, among other invariants:

- magic, version, header size, and complete file size;
- supported UV channel count from one through two;
- every array range and pair of section offsets;
- non-overlapping serialized regions;
- finite vertex attributes and UVs;
- index references and mesh-section ranges;
- material indices and string offsets;
- collision-box storage and values;
- derived axis-aligned bounds.

Vulkan buffers, descriptor sets, and renderer-private handles are never serialized in VMDL.

## Evolution rule

Any VMDL layout change must update the shared format structures, compiler, loader validation, tools, AssetSystem views, and documentation together. Increase the format version when an existing file cannot be interpreted safely under the new contract.
