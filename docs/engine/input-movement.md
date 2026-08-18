# Input & Player Movement

Veil separates raw platform input, action binding, per-tick user commands, and gameplay movement.

## Input path

```text
Platform events
     │
     ▼
Engine SInputState
     │
     ▼
Client input bindings / actions
     │
     ▼
CUserCmdBuilder
     │
     ▼
SUserCommand
     │
     ▼
fixed simulation
     │
     ▼
CPlayerMovementSystem
```

The launcher/platform layer does not directly manipulate the player. It updates platform-independent input state. Client bindings then resolve physical inputs into gameplay actions.

## Bindings and contexts

The client input module contains default bindings, action definitions, action state, contexts, and `CInputBindings`. This allows the same physical key or mouse input to be interpreted according to the active gameplay/UI context rather than hard-wiring platform keys into movement code.

## User commands

`CUserCmdBuilder` converts frame input into an `SUserCommand` assigned to a fixed simulation tick. This creates a clean boundary between variable-rate event collection and deterministic-style simulation consumption.

Mouse deltas, movement intent, button state, and related transient input can be accumulated before a command is consumed by gameplay.

## Player movement

`CPlayerMovementSystem` runs authoritative local-player movement once per fixed tick. It resolves the required ECS state, validates movement tuning, updates view angles, applies Source-style movement behavior, resolves character collision, and commits the resulting state back to ECS.

Movement behavior currently includes:

- yaw/pitch view processing;
- ground friction;
- acceleration and wish movement;
- jumping;
- character gravity;
- direct capsule movement;
- step-move comparison;
- walkable-surface evaluation;
- a short ground probe for stable contact.

Pitch affects the view, while yaw determines horizontal movement orientation.

## Stateless collision queries

Player movement uses PhysicsSystem character queries rather than owning a persistent Box3D character object. Persistent movement state belongs to ECS components. The movement system itself retains only a borrowed physics API reference.

This keeps the movement model independent from the physics backend's internal object lifetime and makes the fixed-tick gameplay state explicit.
