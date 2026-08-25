# Project Structure

## Root

The repository root contains the Premake build definition, project-generation helpers, shader compilation tooling, shared assets, dependency scripts, runtime source, and tool source.

```text
VeilEngine/
├─ assets/
│  └─ shaders/
├─ scripts/
├─ src/
│  ├─ assetsystem/
│  ├─ audiosystem/
│  ├─ client/
│  ├─ engine/
│  ├─ launcher/
│  ├─ physicssystem/
│  ├─ rendersystemvk/
│  ├─ scriptsystem/
│  ├─ shared/
│  └─ tools/
├─ tools/
├─ premake5.lua
├─ GenerateAllProjects.bat
└─ CompileAllShaders.bat
```

## Shared contracts

`src/shared/include` contains public contracts grouped by concern, including API definitions, assets, core utilities, input, interfaces, maps, math, physics, platform abstractions, and rendering structures.

This separation is important: implementation-specific classes can evolve inside their subsystem while the public boundary remains explicit.

The PhysicsSystem implementation keeps Box3D behind that shared boundary and separates its internal responsibilities by ownership:

```text
src/physicssystem/src/
├─ PhysicsSystem.h/.cpp
├─ PhysicsSystemAPI.cpp
├─ body/
│  └─ PhysicsBodyFactory.h/.cpp
├─ character/
│  └─ PhysicsCharacterMover.h/.cpp
├─ scene/
│  └─ PhysicsScene.h/.cpp
└─ shared/
   ├─ Box3DConversions.h/.cpp
   └─ PhysicsValidation.h
```

`CPhysicsSystem` coordinates the public API and scene registry, while each `CPhysicsScene` owns one Box3D world and its scene-local body registry. See [Physics Architecture & Ownership](../engine/physics/architecture.md).

## Shader source

Shader source lives under `assets/shaders`. The current layout separates reusable shader code in `core`, model-specific code in `model`, and editor-specific shaders in `tools`. `StandardMaterial.slang` currently sits at the shader root as a material-facing shader source.

## Tools

`src/tools` currently contains:

```text
assetservices/
mapcompiler/
materialeditor/
modelstudio/
template-tool/
toolcore2/
```

ToolCore2 itself is further separated into application infrastructure, commands, core facilities, FrameGUI, input, platform code, rendering, services, and shared definitions.
