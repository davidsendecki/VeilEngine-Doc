# Veil Engine

Veil Engine is a C++ game-engine project built around a modular runtime, Vulkan rendering, explicit subsystem boundaries, a gameplay/client framework, and compiled asset formats.

!!! warning "Work in progress"
    Veil Engine is under active development. Interfaces, file formats, and subsystem boundaries documented here may change as the engine evolves.

## Purpose of this documentation

This site documents the current runtime architecture and intended usage of Veil Engine. It focuses on subsystem responsibilities, ownership and lifetime rules, data flow, public module APIs, gameplay architecture, rendering, and compiled asset formats rather than attempting to mirror every C++ symbol.

The **VeilEngine repository is the source of truth**. When implementation and documentation disagree, the implementation should be reviewed and the documentation updated accordingly.

## Main areas

- **Architecture** — launcher/platform boundary, dynamic subsystems, API tables, and project organization.
- **Engine** — runtime orchestration, map loading, AssetSystem, PhysicsSystem, and RenderSystemVK.
- **Client & Gameplay** — GameSession, World, entities/components/systems, input, and player movement.
- **Assets & Formats** — asset handles/lifetimes, shaders/materials, VMDL, and VMAP.

## Documentation scope

Tool/editor documentation is intentionally deferred while that architecture is being reworked. Audio and scripting are also outside the current documentation scope. Their presence in the source tree should not be interpreted as missing pages from this runtime-focused documentation pass.

## Documentation maturity

Stable concepts are documented normally; systems undergoing active redesign are explicitly marked as work in progress. The documentation is intended to serve both as a practical reference and as an architecture-review surface while Veil continues to develop.
