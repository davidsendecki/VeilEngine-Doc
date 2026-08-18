# Entities, Components & Systems

Veil's client framework combines class-based gameplay entities with typed component storage and systems. The entity object provides identity/lifecycle/gameplay behavior, while components hold data consumed by reusable systems.

## Entity factory

`CEntityFactory` maps stable human-readable class names to parameterless creation functions. Entity implementations normally register with `REGISTER_ENTITY` during static initialization.

The factory only creates a **detached** `CBaseEntity`. It does not assign a handle, attach the entity to a world, deserialize it, or invoke lifecycle methods. Those responsibilities belong to `CWorld`.

```text
classname
   │
   ▼
CEntityFactory::Create
   │
   ▼
detached CBaseEntity
   │
   ▼
CWorld takes ownership
   │
   ├─ assigns EntityHandle
   ├─ attaches components/properties
   └─ runs lifecycle at the correct stage
```

## Components

The current framework includes typed component infrastructure plus concrete data for areas such as:

- transforms;
- cameras;
- renderables;
- physics;
- player movement/state.

`CComponentRegistry` owns type-erased storage instances, while typed `CComponentStorage<T>` provides storage for each component type.

## Systems

Systems operate on world/component state rather than owning entities themselves. Current examples include:

- `CTransformSystem`;
- `CCameraSystem`;
- `CPhysicsSimulationSystem`;
- `CPlayerMovementSystem`;
- `CRenderExtractionSystem`.

This allows data such as transforms to be shared by multiple concerns without forcing all behavior into a deep entity inheritance tree.

## Entity properties

Compiled map entities carry authored properties. `CEntityPropertyReader` provides the client-side property-reading layer used to convert serialized property data into entity state.

Map construction deliberately creates every entity before applying all properties and before precaching. This makes entity existence and authored data available before lifecycle work that may depend on them.

## Design boundary

Entities and components are **gameplay state**. RenderSystemVK and PhysicsSystem receive engine-defined snapshots, handles, and descriptors rather than taking ownership of the ECS. Backend-specific Vulkan or Box3D objects therefore stay out of the general entity/component model.
