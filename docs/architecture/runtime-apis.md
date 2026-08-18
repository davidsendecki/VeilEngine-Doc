# Runtime API Reference

This page summarizes the public dynamic-module boundaries used by the currently documented runtime. It is intentionally an architectural API reference rather than a replacement for the declarations in `src/shared/include/api`.

## Engine API

`SEngineAPI` is the launcher-facing runtime boundary. Version 1 currently exposes lifecycle, window state, and translated input submission.

### Imports

`SEngineImport` supplies the opaque application window, initial physical drawable extent, shared paths/logger, and the platform mouse-capture callback.

### Exports

The API exposes:

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

The launcher therefore owns event polling while the Engine owns the canonical platform-independent input state.

## AssetSystem API

`SAssetSysAPI` version 1 exposes runtime CPU-asset ownership and dependency batches.

Its main groups are:

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

The AssetSystem import is deliberately small: shared paths and logging.

## RenderSystem API

`SRenderSysAPI` version 1 is the Engine-facing rendering boundary. Its import structure receives the window, paths/logger, and a borrowed AssetSystem API used as the CPU model source during initial GPU upload.

The exported operations are:

```text
Initialize / Shutdown
PrepareModelResources
ClearModelResources
RenderFrame
OnWindowPixelSizeChanged
SetVSync
```

The API reflects the current map-level resource policy: model resources are prepared before activation, and the first implementation can clear the complete model-resource cache during replacement/failure cleanup.

## PhysicsSystem API

`SPhysicSysAPI` is currently version 3. It exposes engine-defined physics handles and descriptors rather than Box3D types.

The API contains scene management, rigid-body operations, velocity/impulse operations, and the stateless `MoveCharacter` query used by player movement.

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

The Game API forms the Engine-to-client boundary. Its implementation is `CClient`, which owns `CGameSession` and the gameplay world beneath it.

The Engine uses this boundary for staged map-world creation/activation, input processing, fixed/frame updates, and render-frame extraction.

## Boundary guideline

These tables should remain relatively narrow. If an implementation class grows a helper method, that does not automatically mean the function belongs in the exported API. Cross-DLL callbacks should represent capabilities genuinely required by the host and preserve subsystem ownership.
