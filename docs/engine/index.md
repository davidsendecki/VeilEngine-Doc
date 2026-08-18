# Engine

The runtime is composed from multiple subsystem modules rather than one monolithic executable implementation. Public contracts are shared through the common include layer, while each subsystem owns its implementation and lifetime-sensitive state.

## Major runtime areas

**Engine** coordinates application-level runtime behavior and subsystem integration. **Client** contains game-facing world behavior. **AssetSystem** owns runtime asset records and loading state. **RenderSystemVK** owns Vulkan rendering. Dedicated modules exist for physics, audio, and scripting.

This modular structure allows systems such as asset loading and rendering to expose narrow APIs while retaining ownership of their internal resources.

## Documentation scope

The initial documentation focuses on the systems that are currently most developed and most relevant to the asset/render pipeline. Other runtime modules will be expanded as their implementations stabilize.
