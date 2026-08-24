# Development Status

Veil Engine is under active development. This page distinguishes **implemented foundations**, **active development**, and **future-facing format/API space** for the systems covered by this documentation.

## Implemented foundations

The documented runtime currently has clear module boundaries for launcher/platform integration, engine orchestration, client/gameplay, runtime assets, physics, and Vulkan rendering.

The runtime asset pipeline has typed handles, dependency batches, CPU model loading, and a separate RenderSystemVK GPU-preparation stage. VMDL version 1 defines a fixed validated static-mesh layout with two interleaved UV slots and model-local collision boxes. VMAP version 1 defines an outer container directory and currently includes the VENT entity-container format.

The client framework has a staged map/world lifecycle, generation-aware entity handles, typed component storage, fixed simulation, render extraction, action-based input, and fixed-tick user commands.

## Active development areas

RenderSystemVK now has documented ownership layers, frame recovery, deferred presentation recreation, managed shaders/pipelines, explicit descriptor contracts, typed resources, and cached GPU models/materials/textures. Skeletal rendering, shadows, additional light types, bindless resources, and runtime material instances remain future work. Public APIs are versioned, but version `1` should not be interpreted as a promise that the overall engine architecture is frozen.

Player movement and physics integration are functional foundations rather than a finished gameplay stack. The current PhysicsSystem API is version 3 and includes the stateless character-movement query used by the client movement system.

## Intentionally deferred documentation

Tool/editor architecture is omitted until its planned rework is far enough along to make documentation useful rather than immediately obsolete. Audio and scripting are also outside the scope of this documentation pass.

## Reserved or future-facing areas

VMDL currently defines static geometry, material paths, bounds, and oriented box collision. Skeletal meshes, animations, richer physics data, and LODs are not part of the current serialized contract.

VMAP's directory-based container model similarly permits additional compiled map-data containers without implying that those containers already exist.

## Documentation policy

Documentation should prefer the following order of authority:

1. current implementation on the `VeilEngine` `main` branch;
2. explicit public contracts and format structures;
3. architecture documentation and comments;
4. planned/future behavior, clearly labeled as such.

When a refactor makes a page inaccurate, update the page rather than preserving obsolete behavior for historical completeness. Git history already provides that history.
