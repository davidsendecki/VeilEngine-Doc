# Game Session & Game Rules

`CGameSession` owns the gameplay world and coordinates its map lifecycle. `CGameRules` provides session policy without owning world state itself.

## GameSession

A session contains:

- the current `CWorld`;
- a `CGameRules` instance;
- the selected active camera;
- the runtime-created local player;
- current started/active state;
- the most recent fixed-tick `SUserCommand`.

The world is declared before GameRules so the borrowed world reference held by the rules object remains valid through destruction.

## Map creation

`CreateMapWorld()` replaces the previous map state with a pending world built from compiled map data and associates that world with its AssetSystem dependency batch.

`PrecacheMapWorld()` asks the world's entities to register their required assets. `ActivateMapWorld()` then spawns/activates the prepared world and selects its gameplay camera.

## GameRules

`CGameRules` is deliberately not an entity. It represents gameplay/session policy and borrows the current world.

The default implementation currently handles local-player creation by finding the first `info_player_start`, creating an unspawned player entity, and transferring the authored spawn transform.

This separation gives future game modes or rulesets a natural customization point without putting global game policy into a map entity.

## Fixed update

Once the session has an active world, `FixedUpdate()` forwards the current `SUserCommand` into fixed gameplay simulation. The command is retained by the session for diagnostics and future controller use.

## Render extraction

`BuildRenderFrame()` asks the active world to produce the camera and render-world data required by RenderSystemVK. The session therefore bridges gameplay ownership and renderer-facing snapshots without transferring ECS ownership to the renderer.
