# Passes, Pipelines & Shaders

RenderSystemVK separates rendering policy, graphics-pipeline ownership, and shader-module ownership. Passes decide what to render, the pipeline manager owns the compatible Vulkan pipelines, and the shader manager owns registered modules.

## Current pass order

Dynamic rendering is active before passes record their commands. The current order is:

1. `CSkyPass` renders the selected sky representation.
2. `ForwardPass` renders queued static meshes with forward PBR shading.

Additional passes should make their attachment, ordering, resource, and synchronization requirements explicit rather than relying on accidental call order.

## Forward pass

`ForwardPass` owns:

- the set-0 draw descriptor layout and pool;
- one draw descriptor set per frame-in-flight slot;
- one persistently mapped uniform buffer per frame-in-flight slot;
- the aligned byte stride between per-draw uniform blocks;
- its static-model vertex-input layout;
- a reusable pipeline-description builder.

The pass borrows `CGPUResourceManager`, `PipelineManager`, the allocator, and the logical device.

During `PrepareFrame()`, it combines camera, lighting, model-transform, and material texture-presence information into one aligned uniform block per queued draw. During `Record()`, it resolves mesh/material resources, binds the managed static forward pipeline, binds the mesh buffers, binds draw and material descriptor sets, and issues indexed draws for the mesh sections.

## Sky pass

`CSkyPass` owns:

- its sky mesh;
- descriptor layout, pool, and per-frame descriptor sets;
- per-frame view and material uniform buffers;
- sky sampler and pass-local fallback textures;
- vertex-input layout and pipeline builder.

It supports panorama and dynamic sky material runtime types. `PrepareFrame()` selects and resolves the current sky material, updates per-frame uniform/descriptor data, and records which pipeline type is prepared for that frame slot.

## Shader manager

`CShaderManager` registers compiled SPIR-V binaries by stable `EShaderType` values. Each shader record contains:

- the compiled filename beneath the configured shader root;
- the Vulkan shader stage;
- the owned `CVkShaderModule`.

The current runtime registry contains the static-model vertex/fragment shaders and the sky vertex, panorama-fragment, and dynamic-fragment shaders.

Shader filenames are confined to registration. Rendering passes and pipeline descriptions refer to stable shader identifiers.

## Pipeline construction

`CGraphicsPipelineBuilder` accumulates a `SManagedGraphicsPipelineDescription`. It copies descriptor layouts, push-constant ranges, and vertex input descriptions into builder-owned storage while configuring:

- shader identifiers;
- color/depth attachment formats;
- topology;
- polygon mode;
- face culling and front-face winding;
- depth testing and writing.

The description borrows the builder-owned arrays and becomes invalid when the builder is modified, reset, moved, or destroyed.

## Pipeline ownership

`PipelineManager` resolves shader identifiers through `CShaderManager`, creates a `CVkGraphicsPipeline`, and owns it under an `EGraphicsPipelineID`.

Current IDs include:

- `ForwardStaticMesh`;
- `SkyPanorama`;
- `SkyDynamic`.

Passes retrieve pipelines by ID without owning or rebuilding their Vulkan objects during command recording.

## Presentation compatibility

Pipelines use dynamic rendering and are created against the current swapchain color format and depth-image format. When presentation recreation changes either attachment format, RenderSystemVK asks the passes to create compatible replacement pipelines through `PipelineManager`.

A resize that preserves both formats does not require pipeline replacement.

## Vertex input

`CVkVertexLayout` owns Vulkan binding and attribute descriptions. `SetBinding()` selects a binding and stride; `AddAttribute()` maps C++ vertex fields to shader input locations.

The static model layout currently exposes position, normal, tangent, UV0, and UV1 directly from the interleaved `SModelVertex` representation.
