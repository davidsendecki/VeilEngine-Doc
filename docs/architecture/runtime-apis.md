# Runtime API Reference

This page summarizes the public dynamic-module boundaries used by the currently documented runtime. It is intentionally an architectural API reference rather than a replacement for the declarations in `src/shared/include/api`.

## Engine API

`SEngineAPI` is the launcher-facing runtime boundary. Version 1 currently exposes lifecycle, window state, and translated input submission.

### Imports

`SEngineImport` supplies the opaque application window, initial physical drawable extent, shared paths/logger, and the platform mouse-capture callback.

### Exports

```text
Initialize / Shutdown
BeginInputFrame
ProcessEngine
OnWindowPixelSizeChanged
SetWindowMinimized
SubmitKeyInput
SubmitMouseButtonInput
SubmitMouseMotion
SubmitMouseWheel
SubmitTextInput
SetInputFocus
```

The launcher owns event polling while the Engine owns the canonical platform-independent input state.

## AssetSystem API

`SAssetSysAPI` version 1 exposes runtime CPU-asset ownership and dependency batches. Model operations use the category-typed `AssetHandle<EAssetType::Model>` handle.

```text
Lifecycle
  Initialize / Shutdown

Dependency batches
  CreateDependencyBatch
  DestroyDependencyBatch
  LoadDependencyBatch
  GetDependencyBatchProgress

Models
  RequestModel
  GetDependencyBatchModelCount
  CopyDependencyBatchModels
  GetModelState
  GetModel
```

The AssetSystem import remains deliberately small: shared paths and logging.

## RenderSystem API

`SRenderSysAPI` version 1 is the Engine-facing rendering boundary. Its import structure receives the window, paths/logger, and a borrowed AssetSystem API used as the CPU model source during GPU preparation.

```text
Initialize / Shutdown
PrepareModelResources
ClearModelResources
RenderFrame
OnWindowPixelSizeChanged
SetVSync
```

The current resource policy prepares model GPU state before map activation and can clear the model-resource cache during replacement/failure cleanup.

## PhysicsSystem API

`SPhysicSysAPI` is currently version 3. It exposes engine-defined physics handles and descriptors rather than Box3D types.

```text
Scenes
  CreateScene / DestroyScene / IsSceneValid / StepScene

Bodies
  CreateBoxBody / DestroyBody / IsBodyValid
  GetBodyTransform / SetBodyTransform
  GetBodyLinearVelocity / SetBodyLinearVelocity
  ApplyBodyImpulse

Character
  MoveCharacter
```

Diagnostic active-scene/body counters are also currently exported.

## Game API

The Game API forms the Engine-to-client DLL boundary. Its implementation is `CClient`, which owns `CGameSession` and the gameplay world beneath it.

### Game imports

`SGameImport` now acts as the capability boundary from the game module back into Engine-owned functionality. It currently contains shared logging/timing, model asset callbacks, PhysicsSystem access, map-load requests, canonical input state, and mouse-capture requests.

Model requests are exposed as Engine capabilities:

```text
Client
  │
  │ RequestModel / GetModel
  ▼
SGameImport
  │
  ▼
Engine
  │
  ▼
AssetSystem
```

The Client therefore does not require a direct `SAssetSysAPI` pointer merely to request or resolve models.

### Game exports

The Engine uses `SGameAPI` for initialization/shutdown, staged map-world creation and activation, input processing, fixed/frame updates, and render-frame extraction.

## Boundary guideline

These tables should remain relatively narrow. If an implementation class grows a helper method, that does not automatically mean the function belongs in an exported API. Cross-DLL callbacks should represent capabilities genuinely required by the host or guest module and preserve subsystem ownership.
