# Tools

Veil includes a dedicated tool ecosystem for authoring and compiling engine content. Tool applications share infrastructure through **ToolCore2** and asset-processing functionality through **AssetServices**.

## Current tool projects

The current `src/tools` tree contains:

- **AssetServices** — shared asset compilation/loading services for tools.
- **Map Compiler** — map compilation tooling.
- **Material Editor** — material authoring, shader inspection, texture workflows, and preview rendering.
- **Model Studio** — model import, authoring, preview, and compilation workflow.
- **Template Tool** — minimal/reference tool project.
- **ToolCore2** — shared application, GUI, input, rendering, command, platform, and service infrastructure.

## Relationship to runtime

Tools should produce assets and metadata consumed by runtime systems, but editor implementation details should not leak into the runtime. Conversely, reusable format definitions and public data contracts belong in shared code when both sides need to understand them.

!!! warning "Tool APIs are evolving"
    ToolCore2 and FrameGUI are active development areas. Documentation for these systems intentionally emphasizes responsibilities and usage concepts over declaring every current API shape permanent.
