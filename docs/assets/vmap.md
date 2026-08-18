# VMAP Map Format

VMAP is Veil's compiled map container format. Unlike a flat entity list, the file is structured as a container with a directory of embedded typed containers.

## Header

VMAP version 1 uses a 32-byte header:

| Field | Purpose |
| --- | --- |
| `Magic[4]` | `VMAP` file signature |
| `Version` | format version, currently `1` |
| `HeaderSize` | serialized header size |
| `ContainerCount` | number of directory entries |
| `Reserved` | reserved for future use |
| `DirectoryOffset` | absolute offset of the container directory |
| `FileSize` | complete VMAP byte size |

The explicit `HeaderSize`, directory offset, and file size give the reader enough information to validate the outer container before interpreting embedded data.

## Container directory

Each embedded container is represented by a 24-byte `VMAPDirectoryEntry`:

```text
ContainerMagic[4]
Flags
Offset
Size
```

`Offset` is absolute from the beginning of the VMAP file and `Size` describes the complete embedded container.

The current flags enumeration defines only `None`; unknown bits are invalid.

## Embedded containers

The format identifies embedded data by four-byte magic rather than hard-coding one monolithic map payload into the outer header.

The current shared format includes a **VENT** container for compiled entity data.

Conceptually:

```text
VMAPHeader
    │
    ▼
Container Directory
    │
    ├─ VENT ── entity data
    ├─ ...
    └─ future containers
```

This allows map data categories to evolve independently while retaining one outer map file.

## Runtime representation

The VMAP reader converts compiled data into the shared `SMapData` representation consumed by the Engine/client map-loading pipeline. The compiled binary layout is therefore kept separate from the runtime object representation.

## Loading relationship

VMAP itself describes the map. Assets referenced by entities are not implicitly ready merely because the VMAP was parsed.

After parsing, the staged loader creates the world, lets entities precache their dependencies, loads CPU assets through AssetSystem, prepares GPU resources through RenderSystemVK, and only then activates gameplay.

!!! warning "Format evolution"
    VMAP is versioned and still under development. New embedded container types should be documented when their binary contract becomes concrete rather than inferred from planned map features.
