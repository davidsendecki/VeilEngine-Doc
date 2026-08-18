# Engine

The runtime is composed from multiple subsystem modules rather than one monolithic executable implementation. Public contracts are shared through the common include layer, while each subsystem retains ownership of its implementation and lifetime-sensitive state.

## Engine role

The **Engine** is primarily an orchestrator. It owns the runtime subsystem loaders/API references, timing and fixed-step scheduling, platform-independent input state, map-load coordination, and the order in which client simulation and rendering are invoked.

It should not absorb responsibilities already owned by another subsystem merely because it coordinates them.

## Documented runtime systems

- **AssetSystem** — CPU-side asset records, typed handles, dependency batches, and model loading.
- **PhysicsSystem** — physics scenes/bodies and character collision queries behind engine-defined handles/descriptors.
- **RenderSystemVK** — Vulkan GPU resources, presentation, and rendering of extracted world snapshots.
- **Client/Game** — gameplay session, world/entities/components, input interpretation, movement, and render extraction.

## Cross-system coordination

Map loading demonstrates the intended architecture well:

```text
Engine / CVMapLoader
      │
      ├─ parse VMAP
      ├─ ask Client to create/precache world
      ├─ ask AssetSystem to load CPU dependencies
      ├─ ask RenderSystemVK to prepare GPU resources
      └─ ask Client to activate gameplay
```

The Engine coordinates the sequence without taking ownership of the internal asset records, Vulkan resources, physics objects, or gameplay ECS.

## Scope

This documentation currently concentrates on the runtime systems above. Other modules can be documented later when their architecture is sufficiently mature and relevant to the runtime reference.
