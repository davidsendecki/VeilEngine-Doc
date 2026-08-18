# Launcher & Platform Boundary

`CLauncher` is the process-level owner of the application window and the outer engine lifecycle. It keeps SDL and platform-facing concerns outside the engine/gameplay modules.

## Responsibilities

The launcher currently owns:

- the SDL application window;
- shared filesystem paths;
- the session logger;
- raw platform event polling;
- platform-independent input state updates;
- physical drawable-size reporting;
- platform mouse-capture application;
- dynamic loading of the Engine subsystem;
- the outer application loop.

## Startup

`main.cpp` constructs the launcher, and `CLauncher::Run()` performs initialization, executes the main loop, and eventually shuts the process down.

The launcher loads the Engine through `CSubsystemLoader<SEngineAPI, SEngineImport>`. It then supplies the services and platform callbacks required by the Engine through the import table.

```text
OS / SDL
   │
   ▼
CLauncher
   │
   ├─ CPaths
   ├─ CLogger
   ├─ SDL_Window
   └─ raw input/platform callbacks
   │
   ▼
SEngineImport
   │
   ▼
Engine DLL / SEngineAPI
```

## Input

SDL events are consumed in the launcher. The resulting state is translated into engine-owned/platform-independent input structures before gameplay sees it.

This means client code does not need to know about SDL scancodes, SDL events, or relative-mouse APIs.

## Window size

The launcher reports the drawable size in **physical pixels**. This is the size needed by RenderSystemVK for swapchain/render-target work and by the client when calculating the camera aspect ratio.

## Mouse capture

The Engine/client can request mouse capture through a callback, but the actual SDL relative-mouse operation remains implemented by the launcher. This is a good example of the platform boundary: higher-level code expresses intent; the platform owner performs the OS/library-specific action.

## Design rule

The launcher should remain thin. Runtime simulation, map loading, rendering policy, and gameplay do not belong in the executable shell merely because it owns the main loop.
