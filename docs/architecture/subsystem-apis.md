# Subsystems & API Tables

Veil's major runtime modules communicate across dynamic-library boundaries using explicit C-style API tables and import structures.

## API and import roles

A subsystem boundary generally has two directions:

```text
Host
 │
 ├─ Import struct ─────────────► Subsystem
 │                              services borrowed by module
 │
 ◄──────────── API table ──────┤
      functions exported by module
```

The **import structure** contains services or callbacks supplied by the host during initialization. The **API table** contains the functions exposed by the loaded module.

Current shared API contracts include Engine, Game, Asset, Physics, and Render interfaces. Audio and scripting contracts also exist in the source tree but are intentionally outside the current documentation scope.

## Why function tables

The module boundary does not expose implementation classes such as `CEngine`, `CPhysicsSystem`, or `CRenderSystemVK` to callers. Instead, the host keeps a pointer to the corresponding API table.

This provides several useful properties:

- implementation classes remain private to their module;
- DLL ownership is explicit;
- the ABI can carry a version number;
- the API structure size can be validated;
- imports make borrowed dependencies visible;
- callers do not delete subsystem implementation objects directly.

## Versioning

`CSubsystemLoader::Validate()` checks both `APIVersion` and `StructSize`. A host can therefore reject an incompatible module before invoking its callbacks.

`ValidateBoundCallbacks()` can additionally validate the callback region of a standard-layout API structure when that region consists only of function pointers.

## Ownership rule

An API pointer returned by a loaded module is **non-owning** and valid only while that dynamic library remains loaded.

Similarly, pointers supplied through an import structure are generally borrowed by the subsystem according to that API's lifetime contract. Initialization code must therefore make ownership and destruction order explicit.

## Dependency direction

A useful example is the runtime stack:

```text
Launcher
   │ loads
   ▼
Engine
   │ loads/co-ordinates
   ├────────► AssetSystem
   ├────────► PhysicsSystem
   ├────────► RenderSystemVK
   └────────► Client/Game
```

The Engine owns the loaders and API-table references for its runtime subsystems. Cross-system operations such as map loading are coordinated at the Engine level rather than allowing the modules to dynamically reach into each other's implementation classes.
