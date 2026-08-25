# Physics

Veil's runtime physics is implemented by the dynamically loaded **PhysicsSystem** subsystem and backed internally by Box3D. Engine and gameplay code communicate through platform-independent descriptors, generation-aware handles, and `SPhysicSysAPI`; Box3D objects never cross the subsystem boundary.

```text
Gameplay entities and components
              │
              ▼
 CPhysicsSimulationSystem
              │ Veil handles and descriptors
              ▼
       SPhysicSysAPI
              │
              ▼
       CPhysicsSystem
              │
              ▼
     CPhysicsScene → Box3D
```

## Current capabilities

The current implementation provides:

- independent physics scenes with configurable gravity;
- static, kinematic, and dynamic rigid bodies;
- primitive box bodies;
- model bodies built from authored VMDL collision boxes;
- per-shape density, friction, restitution, and collision filtering;
- transform and linear-velocity access;
- linear impulses applied at a body's center of mass;
- a stateless capsule query used by client-owned character movement;
- generation-aware scene and body handles with stale- and wrong-scene rejection.

## Ownership summary

`CPhysicsSystem` is the public coordinator. It owns the scene-slot registry and `CPhysicsBodyFactory`. Each occupied scene slot contains a move-only `CPhysicsScene`, which exclusively owns one Box3D world and a scene-local body registry.

On the gameplay side, each `CWorld` owns one `CPhysicsSimulationSystem`. It stores only the opaque scene and body handles returned through `SPhysicSysAPI`, creates bodies from ECS components, pushes kinematic transforms, steps the scene, and pulls simulated dynamic transforms back into ECS state.

See [Architecture & Ownership](physics/architecture.md) for the complete lifetime and responsibility model.

## Fixed simulation

Physics runs only in the fixed simulation path. A world creates pending bodies and publishes kinematic transforms before player movement queries the pre-step scene. The rigid-body scene is then advanced exactly once, dynamic transforms are copied back into ECS storage, transform matrices are updated, and pending entity destruction is flushed.

The character mover never steps the scene itself. It performs a bounded geometric query against the scene's current state and returns backend-neutral contacts and corrected motion to the Client.

See [Fixed-Step Synchronization](physics/fixed-step.md) and [Character Movement](physics/character-movement.md) for the exact order and division of responsibility.

## Documentation map

- [Architecture & Ownership](physics/architecture.md) — module boundaries, internal classes, lifetimes, and destruction.
- [API & Handles](physics/api-handles.md) — imports, callbacks, shared descriptors, and generation validation.
- [Bodies & Collision](physics/bodies-collision.md) — motion types, categories, filtering, materials, and body construction.
- [Fixed-Step Synchronization](physics/fixed-step.md) — ECS body creation and transform flow.
- [Character Movement](physics/character-movement.md) — capsule queries and client-owned movement policy.

## Backend boundary

Box3D identifiers, definitions, shapes, queries, and conversion helpers belong inside `src/physicssystem`. Shared and Client headers use only Veil-owned types. This keeps gameplay independent from the selected physics backend and makes DLL ownership explicit.

!!! warning "Current scope"
    PhysicsSystem does not yet expose general raycasts, shape casts, sensors, overlap events, debug drawing, named surface materials, joints, or runtime collider rebuilding. Primitive sphere and capsule rigid-body descriptors are also not public yet; the existing capsule is used only by the stateless character mover.
