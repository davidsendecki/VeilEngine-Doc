# Rigid Bodies & Collision

Veil separates a rigid body's motion and initial state from the properties of its collision shapes. This avoids duplicating transform, damping, material, and filtering fields for every supported geometry source.

## Bodies and shapes

A body provides the transform and motion state used by the simulation. One or more shapes provide collision geometry, density, surface coefficients, and filtering.

```text
Rigid body
├─ motion type
├─ transform and velocity
├─ damping
└─ one or more collision shapes
   ├─ geometry
   ├─ density
   ├─ friction / restitution
   └─ collision filter
```

One primitive box descriptor creates one body with one shape. One model descriptor creates one body with one shape for every authored collision box.

## Motion types

`EPhysicsMotionType` defines how the body moves:

| Motion type | Behavior |
|---|---|
| `Static` | Immovable body that does not respond to forces or impulses. |
| `Kinematic` | Movable body whose motion is controlled explicitly by gameplay. |
| `Dynamic` | Fully simulated body affected by gravity, contacts, forces, and impulses. |

Motion type is independent from collision category. A kinematic and dynamic body may belong to the same gameplay collision category even though Box3D simulates them differently.

## Body settings

`SPhysicsBodySettings` contains:

| Field | Meaning |
|---|---|
| `MotionType` | Static, kinematic, or dynamic backend motion. |
| `Transform` | Initial position and normalized orientation in scene space. |
| `InitialLinearVelocity` | Linear velocity assigned during creation. |
| `LinearDamping` | Resistance applied to translation. |
| `AngularDamping` | Resistance applied to rotation. |

Scale is not part of `SPhysicsTransform`. Primitive dimensions and model collision scale belong to their geometry descriptors.

## Shape settings

`SPhysicsShapeSettings` applies to every shape produced by one descriptor:

| Field | Meaning |
|---|---|
| `Density` | Contributes to mass calculation for dynamic bodies. |
| `Friction` | Resistance between contacting surfaces. |
| `Restitution` | Bounciness retained after contact. |
| `Filter` | Category membership and accepted collision categories. |

The current descriptor validation requires finite non-negative density and damping, friction and restitution in the inclusive range 0–1, finite transforms, positive box dimensions/scales, and nonempty collision filters.

## Collision categories

`ECollisionCategory` describes what a shape represents for gameplay filtering. It does not describe how the body is simulated.

| Category | Intended role |
|---|---|
| `None` | No category membership. Rejected by current body/query validation. |
| `World` | Environment collision such as map geometry and static objects. |
| `Object` | Movable gameplay objects, including kinematic and dynamic rigid bodies. |
| `Character` | Player-controlled or non-player character collision. |
| `All` | Every category bit. |

Categories can be combined when constructing masks. `SPhysicsCollisionFilter` contains two values:

```cpp
struct SPhysicsCollisionFilter
{
    ECollisionCategory BelongsTo;
    ECollisionCategory CollidesWith;
};
```

A pair is eligible for collision when both filters accept the other shape:

```text
(A.CollidesWith & B.BelongsTo) != None
and
(B.CollidesWith & A.BelongsTo) != None
```

The default character mover belongs to `Character` and queries `World | Object`.

!!! note "Current Client classification"
    The public category system is independent from motion type. The current ECS translator assigns `World` to static and kinematic bodies and `Object` to dynamic bodies. Direct API callers may provide another valid filter. If the Client later classifies kinematic props as `Object`, that is a gameplay policy change rather than a PhysicsSystem ABI change.

## Primitive box bodies

`SPhysicsBoxBodyDesc` combines body settings, shape settings, and local half-extents. The factory validates the descriptor, creates one native body, builds a box hull from the three half-extents, and attaches one hull shape.

If shape creation fails after the native body was created, the factory destroys the partial body before returning an invalid ID.

## Model bodies

`SPhysicsModelBodyDesc` identifies a VMDL model and supplies body/shape settings plus a positive collision scale.

```text
AssetHandle<Model>
        │
        ▼
borrowed SModelView
        │ CollisionBoxes
        ▼
CPhysicsBodyFactory
        │
        ├─ one native body
        └─ one hull shape per collision box
```

Each `SModelCollisionBox` supplies a model-local center, normalized XYZW quaternion, and full width/height/depth. The factory converts the full size to half-widths and creates a scaled oriented box hull.

All collision boxes become shapes on the same body. Model scale is baked into those hulls during creation; it is not retained as a live render-scale link. See [VMDL Collision Boxes](../../assets/vmdl.md#collision-boxes) for the serialized model representation.

The complete collision-box span is validated before the native body is created. If any later shape attachment fails, the partially constructed body and all shapes already attached to it are destroyed together.

## Current mutation rules

The current ECS integration treats collider definitions as creation-time data:

- changing model scale does not rebuild existing shapes;
- changing density, friction, or restitution does not update existing shapes;
- static transforms are not continuously pushed;
- kinematic transforms are replaced before each rigid-body step;
- dynamic transforms are produced by simulation and copied back to ECS.

Callers that need different geometry or material values must currently destroy and recreate the body.
