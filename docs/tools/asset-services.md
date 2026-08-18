# AssetServices

AssetServices is the shared tool-side asset-processing module. It moves compilation and reusable asset I/O out of individual editor applications.

## Current structure

```text
assetservices/src/
├─ ToolServices.cpp/.h
├─ material/
├─ model/
└─ texture/
```

This reflects the intended boundary: editors own authoring UX and documents; AssetServices owns reusable operations on asset data.

## Service API

`ToolServices` exposes the module through the tool service boundary so applications can load the shared functionality rather than statically duplicating compiler implementations.

## Model service

The model area currently contains:

```text
ModelService
VMDLCompiler
VMDLLoader
```

`VMDLCompiler` owns conversion from tool/model data into the compiled VMDL representation. `VMDLLoader` performs the inverse tool-side loading path required when an editor needs to inspect an existing compiled model. `ModelService` exposes those capabilities through the service layer.

## Architectural rule

Model Studio should decide **when** the user wants to compile or load a model, but the binary VMDL implementation should not live in a panel callback.

Likewise, Material Editor can initiate material or texture operations while reusable serialization/compilation belongs in AssetServices.

This keeps asset formats independent from a particular GUI implementation and makes it possible for future tools or command-line workflows to reuse the same processing code.

## Runtime distinction

AssetServices and runtime AssetSystem have different jobs:

| System | Responsibility |
| --- | --- |
| AssetServices | authoring/import/compile/load utilities used by tools |
| AssetSystem | runtime asset requests, handles, dependency batches, CPU asset ownership |
| RenderSystemVK | backend-specific GPU preparation and rendering |

Keeping these boundaries separate avoids turning one "asset system" into a catch-all for editor conversion, runtime lifetime management, and GPU allocation.
