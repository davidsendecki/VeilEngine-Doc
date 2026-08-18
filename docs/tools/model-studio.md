# Model Studio

Model Studio is Veil's model authoring and inspection tool. Its current implementation is organized around application setup, dialogs, importing, model/document logic, panels, rendering, and resources.

## Intended workflow

A typical static-model workflow is:

```text
Source model
    │
    ▼
Importer
    │
    ▼
Model Studio document
    │
    ├─ inspect model
    ├─ assign/edit model data
    └─ preview
    │
    ▼
AssetServices compiler
    │
    ▼
VMDL
```

The important architectural boundary is that **Model Studio owns the authoring experience**, while reusable VMDL compilation/loading logic belongs in **AssetServices**. This allows the same asset-processing implementation to be reused without coupling it to a particular panel or editor window.

## Current focus

The current VMDL runtime format has a concrete static vertex/mesh representation. VMDL's block model already provides room for future skeleton, animation, collision, physics, LOD, and bounds data, but those capabilities should be documented as they become implemented rather than assumed complete from the block enumeration alone.

## Future evolution

Model Studio is expected to grow with the model format. Likely documentation areas include collision authoring, skeletal models, animation inspection, material assignment, model statistics, and compile diagnostics.

!!! warning
    Model Studio and VMDL are active development areas. The workflow documented here describes the current architectural split, not a final editor feature set.
