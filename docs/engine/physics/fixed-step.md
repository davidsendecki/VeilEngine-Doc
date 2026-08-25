# Fixed-Step Physics Synchronization

`CPhysicsSimulationSystem` connects one gameplay `CWorld` to one opaque PhysicsSystem scene. It owns the scene handle but not the underlying Box3D world, ECS components, or pointers into component storage.

## Scene lifetime

The synchronization system creates a scene when the gameplay world is prepared. Replacing or clearing the world first destroys the previous scene and then resets the local handle.

Destroying the PhysicsSystem scene invalidates every body registered inside it. ECS components store only Veil's generation-aware handles; they never retain native body pointers.

## ECS physics components

The current Client uses these responsibilities:

| Component/data | Purpose |
|---|---|
| `SPhysicsBodyComponent` | Motion type, opaque body handle, and body-creation state. |
| `SPhysicsMaterial` | Density, friction, and restitution authored by gameplay. |
| `SBoxColliderComponent` | Primitive box half-extents and material. |
| `SModelColliderComponent` | Model handle whose VMDL collision boxes define the shapes. |
| `STransformComponent` | Authoritative authored transform for static/kinematic bodies and destination for dynamic results. |

An entity with a physics body must have exactly one supported collider source. Supplying neither collider or both box and model colliders is rejected.

## Fixed-tick order

`CWorld::WorldThink()` performs physics and movement in this order:

```text
Process scheduled entity Think functions
                  │
                  ▼
BeginFixedUpdate
  ├─ create pending bodies
  └─ push kinematic transforms
                  │
                  ▼
PlayerMovementSystem::FixedUpdate
  └─ query the pre-step physics scene
                  │
                  ▼
SimulateRigidBodies
  ├─ step the scene once with four solver substeps
  └─ pull dynamic transforms into ECS
                  │
                  ▼
TransformSystem::Update
                  │
                  ▼
FlushDestroyedEntities
```

The order is deliberate. Character movement sees the scene after body creation and kinematic publication but before the current rigid-body step. PhysicsSystem is stepped exactly once regardless of how many character queries occur.

## Pending body creation

At the beginning of a fixed tick, the synchronization system visits live body components:

1. a still-valid public body handle is left unchanged;
2. a stale handle is cleared so creation can be attempted again;
3. a component that has already attempted creation is skipped;
4. otherwise the attempt is latched and the entity is translated into a box or model descriptor.

The latch prevents an invalid authored definition from producing the same error every fixed tick. Explicit body destruction resets the handle and creation state.

Descriptor translation copies the entity's position and normalized rotation, motion type, collider material, dimensions or model handle, and the Client-selected collision filter. PhysicsSystem then performs independent validation before calling Box3D.

## Transform authority

Transform flow depends on motion type:

| Motion type | Before the step | After the step |
|---|---|---|
| Static | Transform supplied during creation. | No synchronization. |
| Kinematic | Current ECS position/rotation pushed to PhysicsSystem. | ECS remains authoritative. |
| Dynamic | PhysicsSystem remains authoritative. | Simulated position/rotation pulled into ECS and marked dirty. |

Kinematic publication currently uses `SetBodyTransform`, which directly replaces the native transform. It is not yet a target-based kinematic movement operation and therefore should not be documented as producing a meaningful kinematic velocity.

## Entity destruction timing

Pending entity destruction is flushed after rigid-body simulation and transform update. An entity marked for destruction during `Think()` can therefore remain represented in the physics scene for the rest of that fixed tick. This is current deliberate ordering, not immediate removal.

Clearing the complete world destroys its physics scene before clearing component storage. Scene destruction releases all native bodies at once and makes any remaining component handles stale.

## Runtime consequences

- Character movement queries dynamic bodies at their pre-step positions.
- The stateless character mover does not apply forces to bodies it touches.
- Moving-platform velocity inheritance is not implemented.
- Runtime collider/material edits require body recreation.
- A successful fixed tick advances a world scene once, even when player step movement performs several character queries.
