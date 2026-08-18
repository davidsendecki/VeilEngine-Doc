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
│  ├─ physicsystem/
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
