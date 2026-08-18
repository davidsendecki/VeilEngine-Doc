# World & Entities

The client-side `CWorld` owns gameplay entities, typed component storage, and the systems that update or extract world state.

## Entity identity

Entities live in reusable world slots. An `EntityHandle` contains slot identity plus a **generation**, allowing the world to reject stale handles after a slot has been reused for a newer entity lifetime.

A slot stores the current entity object, its registered classname, and the generation used to identify that lifetime.

## Entity lifecycle

Map-created entities follow a staged lifecycle rather than being spawned while their serialized data is still incomplete:

```text
Create all entities
      │
      ▼
Deserialize properties
      │
      ▼
Precache all entities
      │
      ▼
CPU/GPU resource preparation
      │
      ▼
Spawn all entities
      │
      ▼
Activate all entities
```

This ensures references and authored properties can exist before `Precache`, and prevents `Spawn`/`Activate` from running before external resources are ready.

For runtime-created entities, `SpawnEntityImmediate()` provides the explicitly named path that precaches, spawns, and activates an entity immediately.

## Components

`CWorld` owns typed component storage through its component registry. Components are attached to generation-aware entities and accessed with typed operations such as `AddComponent<T>`, `FindComponent<T>`, and `HasComponent<T>`.

Component pointers are not intended as permanent identities: storage growth or removal can invalidate direct references. Entity handles remain the stable way to identify an entity lifetime.

## Fixed simulation

`WorldThink()` is the world's fixed-simulation entry point. It runs due entity Think callbacks, flushes deferred destruction, processes simulation systems, and refreshes dirty transforms.

Think scheduling uses absolute simulation ticks and a schedule revision. Old queue entries can therefore be discarded when a newer schedule supersedes them.

## Deferred destruction

Calling `DestroyEntity()` makes the handle non-living immediately, but reclamation of the entity object and components is deferred until the world's destruction flush. This avoids tearing down storage in the middle of unrelated simulation work.

## Rendering boundary

The world does **not** issue renderer commands directly. Rendering consumes self-contained `SRenderView` and `SRenderWorld` snapshots extracted from gameplay state. This keeps simulation ownership separate from backend rendering execution.
