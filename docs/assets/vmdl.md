# VMDL Model Format

VMDL is Veil's compiled model format. The current format is versioned and block-based so model data can be separated by concern and extended over time.

!!! warning "Format under development"
    VMDL is still evolving. Do not assume the current version is a permanent compatibility contract.

## File header

The current `VMDLHeader` is 16 bytes and contains:

| Field | Purpose |
| --- | --- |
| `Magic[4]` | `VMDL` file signature |
| `Version` | format version; currently `1` |
| `BlockCount` | number of block-directory entries |
| `FileSize` | expected complete file size |

## Block directory

Each block is described by a 24-byte `VMDLBlockEntry` containing its type, block version, file offset, and byte size.

The block type enumeration currently reserves or defines the following logical areas:

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

Not every listed block type must be implemented or emitted by the current static-model pipeline. The enumeration also establishes room for later model capabilities.

## Static mesh block

The current mesh block starts with `VMDLMeshBlockHeader`, containing the mesh count. Each renderable mesh is then serialized as:

```text
VMDLMesh
VMDLVertex[VertexCount]
uint32_t[IndexCount]
```

`VMDLMesh` stores a name string-table index, material index, vertex/index counts, and local bounds.

The first runtime vertex layout is 48 bytes per vertex and contains:

```text
Position : float3
Normal   : float3
TexCoord : float2
Tangent  : float4
```

## Material references

Meshes reference materials by **index**, not by embedding a complete material path in every mesh. String data is handled separately through the format's string-table facilities.

## Runtime loading

`CAssetSystem` uses `CVMDLReader` to load compiled model data into runtime model records. Runtime consumers access the result through typed model handles and borrowed model views rather than directly owning the reader's internal representation.

## Future blocks

The block architecture already names skeleton, animation, collision, physics, LOD, and bounds areas. Their presence in the type enumeration should be read as format architecture/reserved capability, not as a guarantee that the corresponding runtime pipeline is complete today.
