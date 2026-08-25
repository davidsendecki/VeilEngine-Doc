# Physics Architecture & Ownership

PhysicsSystem separates public orchestration, native-world ownership, body construction, gameplay synchronization, and character movement. The separation keeps Box3D private while allowing the Client to control gameplay policy.

## Ownership tree

```text
Engine
└─ subsystem loader
   └─ borrowed SPhysicSysAPI

CWorld
└─ CPhysicsSimulationSystem
   └─ SPhysicsSceneHandle
      └─ SPhysicsBodyHandle values in ECS components

CPhysicsSystem
├─ CPhysicsBodyFactory
└─ scene slot registry
   └─ CPhysicsScene
      ├─ b3WorldId
      └─ scene-local body slot registry
         └─ b3BodyId
```

The Engine owns the dynamic-library loader and the borrowed API-table reference. `CWorld` owns the gameplay-facing synchronization object. PhysicsSystem owns every Box3D object.

## Responsibility boundaries

| Owner | Responsibility |
|---|---|
| Engine | Loads PhysicsSystem, validates its API table, supplies imports, and drives fixed ticks through the Client. |
| `CWorld` | Owns the world-scoped `CPhysicsSimulationSystem` and determines fixed-tick ordering. |
| `CPhysicsSimulationSystem` | Translates ECS components into descriptors and synchronizes kinematic/dynamic transforms. |
| `CPhysicsSystem` | Validates public operations, owns public scene slots, resolves scenes, coordinates body creation, and exposes the API behavior. |
| `CPhysicsScene` | Exclusively owns one Box3D world, its local body slots, free list, generations, and active-body count. |
| `CPhysicsBodyFactory` | Validates descriptors and creates native bodies and shapes in a supplied world. |
| `B3DUtil` | Converts Veil math/filter types and performs backend-specific validation and character queries. |
| AssetSystem | Owns model data and lends `SModelView` values through the Engine-provided resolver. |

## Public scene registry

`CPhysicsSystem` stores reusable scene slots. Each slot contains:

- a move-only `CPhysicsScene`;
- a scene generation;
- an occupied flag.

Destroying a scene shuts down its native world, clears its local body registry, advances the scene-slot generation, and returns the slot index to the free list. A stale `SPhysicsSceneHandle` therefore cannot resolve a replacement scene that reuses the same index.

`CPhysicsScene` moves its world, body slots, free list, and active-body count together. This is required because growth of the outer scene vector can move a live scene after bodies have already been registered.

## Scene-local body ownership

Each scene maintains its own reusable body slots rather than placing every body in one global PhysicsSystem array. A slot contains a native `b3BodyId`, a body generation, and an occupied flag.

The public body handle embeds the complete owning scene identity in addition to the scene-local body index and generation. Resolution validates all of these before returning a native body:

1. the supplied scene handle resolves to a live scene;
2. the body handle is structurally valid;
3. the body handle's embedded scene equals the supplied scene;
4. the local body index is in range;
5. the slot is occupied and its generation matches;
6. Box3D still reports the native body as valid.

This prevents a body from Scene A from aliasing a coincidentally identical local slot in Scene B.

## Body creation flow

```text
Client descriptor
      │
      ▼
CPhysicsSystem
  resolve scene
  resolve model view when required
      │
      ▼
CPhysicsBodyFactory
  validate descriptor
  create native body
  attach native shapes
      │
      ▼
CPhysicsScene::RegisterBody
  allocate/reuse local slot
  return public body handle
```

The native body remains the caller's responsibility until registration succeeds. If the factory creates a body but scene registration fails, `CPhysicsSystem` destroys that native body immediately. After registration, the scene owns its lifetime.

## Model-data boundary

The Engine supplies PhysicsSystem with a synchronous `GetModel` callback. `CPhysicsSystem::CreateModelBody()` resolves a borrowed `SModelView`, passes its collision-box span directly to the factory, and returns before the view's borrowed storage can expire.

Neither `CPhysicsSystem`, `CPhysicsBodyFactory`, nor `CPhysicsScene` caches the view or its spans. AssetSystem retains ownership of the underlying model data.

## Destruction order

Destroying one body destroys its native Box3D body, releases the local slot, and advances the body generation. Destroying an entire scene destroys the Box3D world once and then clears the local registry; it does not destroy each body immediately before destroying the same world.

Shutdown clears all scene slots. Destructors and move assignment use the same idempotent scene shutdown path, preventing double destruction of worlds or bodies.

## Architectural rules

- Box3D types remain private to `src/physicssystem`.
- Public handles are runtime identities, not serialized asset or map data.
- `CPhysicsScene` owns native worlds and body registries; it does not resolve assets.
- `CPhysicsBodyFactory` constructs native bodies; it does not allocate public handles.
- The Client owns ECS and gameplay policy; PhysicsSystem does not traverse `CWorld`.
- Every valid world scene is stepped exactly once per fixed tick.
- Character movement queries are stateless and never advance the world.
