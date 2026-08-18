# Architecture Overview

Veil Engine is split into relatively independent runtime subsystems and a separate tool ecosystem. Shared public contracts live in the `shared` project so modules can communicate without exposing their internal implementation.

## Runtime modules

The current source tree contains dedicated modules for:

- `engine` — runtime orchestration and engine-level services.
- `client` — game/client-side world and gameplay integration.
- `assetsystem` — runtime asset ownership, loading, handles, and dependency batches.
- `rendersystemvk` — Vulkan rendering backend.
- `physicsystem` — physics integration.
- `audiosystem` — audio subsystem.
- `scriptsystem` — scripting subsystem.
- `launcher` — executable startup layer.
- `shared` — public APIs, interfaces, asset structures, math, map structures, and common contracts.

## Tool ecosystem

Editor applications are kept below `src/tools`. Shared editor infrastructure is provided by **ToolCore2**, while asset processing is separated into **AssetServices**. Current applications include the **Material Editor**, **Model Studio**, and **Map Compiler**.

## Architectural direction

The intended boundary is that high-level engine and tool code should depend on subsystem APIs and stable data contracts rather than directly reaching into backend implementation details. This is especially important for rendering: Vulkan-specific resource and synchronization details belong below the renderer's higher-level interfaces.

!!! note "Living architecture"
    This documentation describes the implementation as it exists now and the direction visible in the codebase. Components marked as work in progress should not yet be treated as permanent API contracts.
