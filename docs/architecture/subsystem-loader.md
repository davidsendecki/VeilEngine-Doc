# SubsystemLoader

`CSubsystemLoader<TAPI, TImport>` is Veil's generic dynamic-library loader for subsystem API tables.

It is infrastructure for locating and validating a module boundary; it is **not** the subsystem itself.

## Load sequence

`LoadSubsystem()` performs the following sequence:

```text
Unload previous module
       │
       ▼
Build platform library name
       │
       ▼
Load dynamic library
       │
       ▼
Resolve exported API getter
       │
       ▼
Call getter
       │
       ▼
Store API table pointer
```

On Windows the loader builds a `.dll` filename. Other supported platform code paths use the corresponding shared-library naming convention.

## ABI validation

After loading, the host can call `Validate(ExpectedVersion)`. Validation checks:

1. an API table was actually returned;
2. `APIVersion` matches the host's expected version;
3. `StructSize` is at least as large as the API structure expected by the host.

This catches a class of binary mismatches before the host starts calling into the module.

## Callback validation

For suitable standard-layout API tables, `ValidateBoundCallbacks()` treats the callback portion as a contiguous function-pointer block and verifies that every callback is bound.

This is separate from ABI validation: a structure can have the correct version and size while still containing a null function pointer.

## What it does not own

The loader owns the **dynamic-library handle**, but it does not create or store the subsystem implementation instance.

That distinction is important. A module can internally expose a process-wide implementation behind its API getter while the host sees only the stable table of callbacks.

## Unloading

`Unload()` clears the API/getter pointers and releases the dynamic-library handle. It is safe to call repeatedly, and the loader destructor invokes it automatically.

Any API-table pointer obtained from `GetAPI()` must therefore be considered invalid after unload.

## Usage pattern

```text
CSubsystemLoader<TAPI, TImport>
        │
        ├─ LoadSubsystem(...)
        ├─ Validate(API_VERSION)
        ├─ ValidateBoundCallbacks(...)
        │
        ▼
const TAPI* pAPI = GetAPI()
        │
        ▼
Initialize(imports)
        │
        ▼
normal subsystem calls
        │
        ▼
Shutdown()
        │
        ▼
Unload()
```

The exact Initialize/Shutdown callbacks belong to each subsystem API; the loader only provides the generic module-loading machinery.
