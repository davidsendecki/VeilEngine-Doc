# Client & Gameplay Architecture

The `client` module is Veil's Engine-facing gameplay module. It owns gameplay/session state and exposes that state to the Engine through `SGameAPI` rather than exposing `CWorld` or entity implementation classes across the DLL boundary.

## Ownership hierarchy

```text
Engine
  │ owns module loader / borrows SGameAPI
  ▼
CClient
  │ owns
  ▼
CGameSession
  │
  ├─ CWorld
  ├─ CGameRules
  ├─ local player handle
  └─ active camera handle
```

Engine services supplied through the game import table are borrowed for the client/session lifetime. Gameplay does not take ownership of AssetSystem, PhysicsSystem, the Engine clock, or platform state.

## CClient boundary

`CClient` translates the exported pointer-based module API into the reference/object-oriented session interface. Its main responsibilities are staged map-world operations, input resolution, fixed/frame update dispatch, and renderer-facing snapshot extraction.

## Map lifecycle

The client participates in map loading without owning the complete loading pipeline:

```text
CreateMapWorld
      │
      ▼
PrecacheMapWorld
      │
      │ Engine loads CPU/GPU dependencies
      ▼
ActivateMapWorld
```

`AbortMapWorld` clears pending/active map-specific state during replacement or failure.

The Engine remains the coordinator because the complete operation crosses VMAP parsing, gameplay, AssetSystem, and RenderSystemVK.

## Simulation and input

Raw platform input is translated before it reaches gameplay. Client input bindings resolve semantic actions once per rendered frame, while `CUserCmdBuilder` produces an `SUserCommand` for each fixed simulation tick.

The active session consumes that command during `FixedUpdate`. This keeps variable-rate event collection separate from fixed-rate gameplay simulation.

## Render extraction

Gameplay owns the ECS; the renderer does not. `BuildRenderFrame()` produces self-contained `SRenderView` and `SRenderWorld` snapshots for the current frame.

This boundary allows rendering architecture to evolve without making Vulkan code depend on client entity/component storage.

## Platform independence

Client code requests operations such as map loading or mouse capture through Engine-provided callbacks. SDL and launcher-specific implementation remains outside the gameplay module.
