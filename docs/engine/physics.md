# Physics

Veil exposes physics through a dedicated PhysicsSystem module backed by Box3D. Gameplay code communicates through `SPhysicSysAPI` and engine-defined handles/descriptors rather than retaining Box3D objects directly.

## PhysicsSystem responsibilities

`CPhysicsSystem` currently owns physics scenes and rigid bodies, steps scenes, exposes body transforms and linear velocity, applies impulses, and provides character movement queries.

Both scenes and bodies use generation-aware handles backed by reusable slots. Destroyed slots can be recycled without making an old handle accidentally resolve to a newer object.

## Scene ownership

A physics scene represents one simulation world. Bodies are associated with the scene that created them, and destroying a scene invalidates its bodies.

The client-side world uses a `CPhysicsSimulationSystem` to bridge ECS physics components and the PhysicsSystem API. This keeps Box3D-specific ownership inside the physics module while gameplay retains its own entity/component representation.

## Fixed stepping

Physics simulation is driven from the engine's fixed simulation path. The player movement system performs its character collision queries before the world's rigid-body scene receives its single fixed step.

This ordering is deliberate: character movement does not independently advance the physics world.

## Character mover

The PhysicsSystem exposes `MoveCharacter` using engine-owned `SCharacterMoverDesc`, `SCharacterMoveInput`, and `SCharacterMoveResult` structures. The player movement system can therefore perform capsule movement/collision queries without exposing Box3D types to gameplay code.

## Boundary rule

Gameplay and engine systems should prefer the public physics API. Box3D identifiers and implementation details belong inside `physicsystem`; leaking them into ECS components would couple gameplay state to the selected physics backend.
