# Architecture Overview

Veil Engine is split into relatively independent runtime subsystems. Shared public contracts live in the `shared` project so modules can communicate without exposing their internal implementation.

## Documented runtime modules

The current documentation focuses on:

- `launcher` — executable startup and platform boundary.
- `engine` — runtime orchestration and engine-level services.
- `client` — game/client-side world and gameplay integration.
- `assetsystem` — runtime asset ownership, loading, handles, and dependency batches.
- `rendersystemvk` — Vulkan rendering backend.
- `physicsystem` — physics integration.
- `shared` — public APIs, interfaces, asset structures, math, map structures, and common contracts.

Other experimental or not-yet-priority runtime modules may exist in the repository but are intentionally omitted from this documentation until their architecture is ready to be treated as useful reference material.

## Tools

The repository also contains a substantial editor/tool ecosystem. Tool documentation is intentionally deferred while that architecture is being reworked. The runtime documentation should therefore not be read as an inventory of every project present in the repository.

## Architectural direction

High-level engine and gameplay code should depend on subsystem APIs and stable data contracts rather than directly reaching into backend implementation details. This is especially important for rendering and physics: Vulkan and Box3D-specific resource/lifetime details belong behind their subsystem boundaries.

Cross-subsystem workflows are coordinated by an owner at the appropriate level. Map loading is a representative example: the Engine coordinates parsing, client world creation, AssetSystem CPU loading, RenderSystemVK GPU preparation, and final world activation rather than allowing those modules to take ownership of one another.

!!! note "Living architecture"
    This documentation describes the implementation as it exists now and the direction visible in the codebase. Components marked as work in progress should not yet be treated as permanent API contracts.
