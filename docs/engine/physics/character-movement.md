# Character Movement

Veil's player is not represented as a dynamic rigid body. The Client owns Source-style gameplay movement and asks PhysicsSystem to resolve individual translations for a vertical capsule.

This division keeps acceleration, jumping, grounding, stepping, and view behavior under game control while PhysicsSystem retains exclusive access to Box3D queries and collision-plane solving.

## Responsibility split

| Client / `CPlayerMovementSystem` | PhysicsSystem / `B3DUtil` |
|---|---|
| Interprets `SUserCommand`. | Builds the native capsule. |
| Applies ground friction and acceleration. | Performs mover casts and overlap/collision queries. |
| Applies air acceleration and gravity. | Converts query contacts into bounded Veil contacts. |
| Handles jump press edges. | Solves collision planes and clips velocity. |
| Chooses direct or step movement. | Applies collision filtering. |
| Classifies walkable ground and steep slopes. | Returns corrected translation and velocity. |
| Writes player and camera transforms. | Never steps the scene or owns gameplay state. |

## Public query data

`SCharacterMoverDesc` describes a vertical capsule using a radius, total height, and collision filter. The position convention is always the capsule's **feet origin**.

```text
feet origin
    │
    ├─ lower sphere center at Radius
    └─ upper sphere center at Height - Radius
```

`SCharacterMoveInput` contains:

- current feet position;
- desired translation for this query;
- velocity to be clipped by collision response.

`SCharacterMoveResult` returns:

- resolved feet position;
- translation actually applied;
- corrected velocity;
- up to `MAX_CHARACTER_CONTACTS` contact points and normals.

`MAX_CHARACTER_CONTACTS` is currently 16. It is a fixed DLL-safe capacity, not the number of contacts every query produces. `ContactCount` identifies the initialized entries.

## Query pipeline

For one `MoveCharacter` call, PhysicsSystem:

1. validates the scene, capsule geometry, input vectors, and collision filter;
2. builds a Box3D capsule from the feet-origin convention;
3. casts the mover along the desired translation;
4. gathers collision planes at the resulting position;
5. solves those planes to produce a nonpenetrating translation;
6. clips the supplied velocity against the resolved collision planes;
7. converts the bounded contact data back into Veil-owned structures.

The query is synchronous and stateless. It does not create a rigid body, retain a capsule, cache contact spans, or call `StepScene()`.

## Client movement sequence

The Client performs movement during the fixed tick:

```text
SUserCommand
    │
    ├─ update view angles
    ├─ calculate wish direction
    ├─ ground friction / acceleration or air acceleration
    ├─ jump edge and gravity
    └─ desired translation
             │
             ▼
       MoveCharacter
             │
             ├─ evaluate direct movement
             ├─ optionally evaluate up / forward / down step path
             └─ probe and classify ground
```

The step path is accepted only when it finds walkable ground and produces better horizontal progress than the direct result. Grounding is based on contact normals and the configured maximum slope angle; steep surfaces remain walls rather than becoming walkable ground.

## Collision filtering

The default character filter is:

```cpp
SPhysicsCollisionFilter
{
    ECollisionCategory::Character,
    ECollisionCategory::World | ECollisionCategory::Object
};
```

The capsule therefore queries world and object shapes. Contacting a dynamic object affects the character's resolved translation, but the current mover does not apply an equal physical impulse to that object.

## Fixed-step relationship

Character movement runs after pending rigid bodies and kinematic transforms have been published, but before the scene's current rigid-body step. Multiple calls may be made during one player update for direct movement, step evaluation, and ground probing; none of them advance time.

Render-only frames do not perform movement or physics. Fixed-tick input edges such as jumping are consumed by the Client rather than being repeated by the stateless query layer.

## Current limitations

- The character does not physically push dynamic bodies.
- Moving-platform velocity inheritance is not implemented.
- The mover is an upright capsule and does not rotate with the player view.
- The public result has a fixed contact capacity rather than an unbounded collection.
- General-purpose public capsule casts and overlap queries are not exposed separately yet.
