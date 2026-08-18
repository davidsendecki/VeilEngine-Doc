# Engine Lifecycle

`CEngine` is the central runtime coordinator. The launcher owns the process-wide engine instance and supplies platform services, the native window, and translated input.

## Initialization

Initialization validates launcher imports and then loads the required runtime modules through `CSubsystemLoader`. The engine currently owns loaders/API tables for:

- AssetSystem
- RenderSystemVK
- PhysicsSystem
- AudioSystem
- game/client module
- ScriptSystem

Subsystem initialization is deliberately centralized so dependency order and shutdown behavior remain explicit.

## Frame processing

One call to `ProcessEngine()` represents one engine frame. The internal structure is:

```text
ProcessEngine
    │
    ├─ BeginFrame
    │    └─ update timing
    │
    ├─ RunFixedSimulation
    │    └─ consume fixed-step accumulator
    │
    ├─ FrameUpdate
    │    └─ variable-rate client/subsystem work
    │
    ├─ RenderFrame
    │    └─ submit current frame to renderer
    │
    ├─ EndFrame
    │    └─ frame statistics
    │
    └─ ProcessPendingMapLoads
         └─ safe-boundary map replacement
```

The separation between fixed simulation and variable-rate frame work is intentional. Simulation can advance in stable fixed increments while rendering and presentation follow the actual frame rate.

## Timing

`CEngine` owns `CEngineTime`, a fixed-step accumulator, and FPS accounting. The engine clock is shared with systems such as the client world rather than each subsystem independently inventing its own simulation timeline.

## Input boundary

The launcher translates platform events into engine-level input operations such as key state, mouse buttons, mouse motion, wheel input, text input, and focus changes. This keeps platform event handling outside gameplay code.

## Deferred map requests

The client does not directly replace the active map while arbitrary frame work is running. Map names requested through the client import API are copied into a FIFO queue and consumed by `ProcessPendingMapLoads()` at a safe frame boundary.

This is an important ownership rule: the engine coordinates the transition because map loading spans the client world, assets, and renderer.
