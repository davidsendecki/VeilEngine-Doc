# Physics API & Handles

PhysicsSystem is a dynamic subsystem. Its cross-DLL contract is defined by `SPhysicSysImport`, `SPhysicSysAPI`, and the backend-neutral structures under `src/shared/include/physics`.

The current PhysicsSystem API version is **1**.

## Imports

`SPhysicSysImport` supplies the services PhysicsSystem borrows during initialization:

| Import | Contract |
|---|---|
| `pLogger` | Non-owning pointer to the shared logging service. |
| `GetModel` | Synchronous callback that resolves a ready model into a borrowed `SModelView`. |

The model view and its spans are valid only for the synchronous operation that requested them. PhysicsSystem consumes model collision data during `CreateModelBody()` and never retains the view.

## Exported operations

`SPhysicSysAPI` groups its callbacks by responsibility:

```text
Lifecycle
  Initialize / Shutdown

Scenes
  CreateScene / DestroyScene
  IsSceneValid / StepScene

Rigid bodies
  CreateBoxBody / CreateModelBody
  DestroyBody / IsBodyValid
  SetBodyTransform / GetBodyTransform
  SetBodyLinearVelocity / GetBodyLinearVelocity
  ApplyBodyImpulse

Character
  MoveCharacter

Diagnostics
  GetActiveSceneCount / GetActiveBodyCount
```

The diagnostic counters exist for lifecycle verification and debugging. They are not gameplay ownership mechanisms.

## Scene handles

`SPhysicsSceneHandle` contains a reusable slot index and generation:

```cpp
struct SPhysicsSceneHandle
{
    uint32_t Index;
    uint32_t Generation;
};
```

Default construction produces an invalid handle. `IsValid()` performs a structural check only: the index must not be the invalid sentinel and the generation must be nonzero. Live validation still requires `IsSceneValid()` or another scene operation inside PhysicsSystem.

When a scene slot is destroyed and reused, its generation advances. An outstanding handle with the previous generation remains stale.

## Body handles

Bodies use a scene-qualified identity:

```cpp
struct SPhysicsBodyHandle
{
    SPhysicsSceneHandle Scene;
    uint32_t Index;
    uint32_t Generation;
};
```

`Scene` is the complete owning scene identity, including its generation. `Index` and `Generation` identify a reusable body slot local to that scene.

Public body operations intentionally receive both a scene handle and a body handle:

```cpp
Physics.IsBodyValid(Scene, Body);
Physics.SetBodyTransform(Scene, Body, &Transform);
Physics.DestroyBody(Scene, Body);
```

PhysicsSystem does not silently replace the supplied scene with `Body.Scene`. The two identities must match. This makes wrong-scene calls explicit failures and protects scenes that happen to contain the same local body index/generation.

## Shared descriptors

The public structures separate body state from shape state:

| Type | Purpose |
|---|---|
| `SPhysicsSceneDesc` | Settings used to create a scene, currently including gravity. |
| `SPhysicsTransform` | Position and normalized rotation in physics-scene space. |
| `SPhysicsBodySettings` | Motion type, initial transform, velocity, and damping. |
| `SPhysicsShapeSettings` | Density, friction, restitution, and collision filter. |
| `SPhysicsBoxBodyDesc` | Body and shape settings plus local box half-extents. |
| `SPhysicsModelBodyDesc` | Model handle, body/shape settings, and collision scale. |
| `SCharacterMoverDesc` | Vertical capsule geometry and collision filter. |
| `SCharacterMoveInput` | Feet origin, desired translation, and velocity. |
| `SCharacterMoveResult` | Applied motion, corrected velocity, and fixed-capacity contacts. |

Shared contracts remain platform-independent and use fixed-layout values suitable for the DLL boundary. They do not contain STL containers, callbacks with ownership ambiguity, or Box3D identifiers.

## Validation and failure behavior

Public creation functions return invalid handles on failure. Queries and mutations return `false` when their scene, body, output pointer, or supplied data is invalid. Destroy operations safely reject stale and wrong-scene handles.

Validation is intentionally layered:

- Client translation validates authored ECS data and produces contextual entity errors;
- `CPhysicsSystem` validates pointers and public handles;
- `CPhysicsBodyFactory` validates descriptor values before calling Box3D;
- `CPhysicsScene` validates local body ownership and generations.

This duplication at subsystem boundaries prevents a new or faulty caller from passing invalid values directly into the backend.

## ABI rule

Changing the layout of an imported/exported API table, handle, or by-value descriptor requires an API-version review and a clean rebuild of Shared, PhysicsSystem, Engine, and Client. Source-only helper changes inside `src/physicssystem` do not automatically require a public API bump.
