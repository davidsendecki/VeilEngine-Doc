# Client / Game Architecture

The `client` module is Veil's engine-facing gameplay module. `CClient` owns the active `CGameSession` and translates the pointer-based exported `SGameAPI` boundary into the session's gameplay-facing interface.

## Ownership

The engine owns the client module itself, while `CClient` owns all gameplay state beneath the active session. Engine services such as logging, timing, assets, and physics are borrowed through the game import API and must outlive the client.

```text
Engine
  │
  ▼
SGameAPI / CClient
  │
  ▼
CGameSession
  │
  ├─ CWorld
  ├─ CGameRules
  ├─ local player
  └─ active camera
```

## Map lifecycle

The client participates in the staged map-loading pipeline through four operations:

```text
CreateMapWorld
     │
     ▼
PrecacheMapWorld
     │
     ▼
ActivateMapWorld

AbortMapWorld ── used for failure/replacement
```

This keeps the engine's cross-subsystem map loader in control while the client owns creation and activation of gameplay state.

## Runtime updates

`ProcessInput()` resolves raw engine input once per rendered frame. `FixedUpdate()` advances gameplay using a command associated with a fixed simulation tick. `FrameUpdate()` is reserved for frame-rate-dependent client work.

## Rendering boundary

`BuildRenderFrame()` extracts `SRenderView` and `SRenderWorld` snapshots from the active session. The renderer receives those snapshots; it does not traverse client entities or components directly.

## Platform boundary

The client can request map loading and mouse capture through callbacks supplied by the engine. It does not directly depend on SDL or other launcher/platform APIs.
