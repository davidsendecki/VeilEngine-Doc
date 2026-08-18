# Development Status

Veil Engine is under active development. This page exists to make a distinction between **implemented architecture**, **work in progress**, and **future-facing format/API space**.

## Implemented foundations

The repository currently contains dedicated runtime modules for assets, audio, client/game integration, engine orchestration, physics, Vulkan rendering, scripting, shared contracts, and startup. The tool suite contains ToolCore2, AssetServices, Material Editor, Model Studio, Map Compiler, and a template tool.

The asset pipeline has a concrete typed-handle model and runtime model loading path. VMDL version 1 defines a block directory and a concrete static-mesh representation. Shader source is organized around Slang modules with shared core functionality and tool-specific preview shaders.

## Active development areas

The Vulkan renderer, ToolCore2 rendering infrastructure, FrameGUI, Model Studio, and the broader model pipeline are evolving. Their documentation should be reviewed whenever a substantial refactor lands.

## Reserved or future-facing areas

VMDL currently names block types for skeletons, animations, collision, physics, LODs, and bounds. These entries provide architectural space but should not be interpreted as proof that the complete corresponding authoring/runtime feature is implemented.

## Documentation policy

Documentation should prefer the following order of authority:

1. current implementation on the `VeilEngine` main branch;
2. explicit public contracts and format structures;
3. architecture documentation and comments;
4. planned/future behavior, clearly labeled as such.

When a refactor makes a page inaccurate, update the page rather than preserving obsolete behavior for historical completeness. Git history already provides that history.
