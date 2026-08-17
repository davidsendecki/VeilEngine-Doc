# Veil Engine

Veil Engine is a C++ game-engine project with a modular runtime, a Vulkan rendering backend, an asset pipeline, and a dedicated suite of development tools.

!!! warning "Work in progress"
    Veil Engine is under active development. Interfaces, file formats, tools, and subsystem boundaries documented here may change as the engine evolves.

## Purpose of this documentation

This site documents the architecture and intended usage of the engine and its tools. It focuses on subsystem responsibilities, data flow, high-level APIs, asset formats, and development workflows rather than attempting to provide an exhaustive reference for every C++ symbol.

The **VeilEngine repository is the source of truth**. When implementation and documentation disagree, the implementation should be reviewed and the documentation updated accordingly.

## Main areas

- **Engine** — runtime systems and services.
- **Rendering** — Vulkan renderer and graphics architecture.
- **Assets & Formats** — runtime assets, shaders, materials, and compiled formats.
- **Tools** — ToolCore2, FrameGUI, Material Editor, Model Studio, and related editor infrastructure.
- **Architecture** — module boundaries and project organization.

## Documentation maturity

This is the first documentation pass. Stable concepts are documented normally; systems undergoing active redesign are explicitly marked as work in progress. This makes the documentation useful as both a reference and an architecture-review surface while Veil continues to develop.
