# Rendering

Veil's runtime rendering backend is implemented in `src/rendersystemvk` and targets Vulkan.

!!! warning "Active development"
    The rendering architecture is undergoing active development. Treat the current class and folder boundaries as implementation state rather than a frozen public API.

## Layering

The renderer source is currently divided into several concerns:

```text
rendersystemvk/src/
├─ backend/
├─ commands/
├─ core/
├─ presentation/
├─ render/
├─ shared/
└─ vulkan/
```

The separation reflects an important design goal: Vulkan mechanics should remain below higher-level rendering concepts. Game and feature code should not need to manually reproduce Vulkan resource creation, synchronization, presentation, or pipeline setup.

## Vulkan layer

The low-level Vulkan side owns the API-facing objects and helpers required to operate the device. Higher layers can then build rendering behavior using those primitives without scattering raw setup code throughout the renderer.

## Rendering layer

The `render` area is intended for higher-level rendering concepts such as render queues, passes, shader management, and graphics pipelines. A render pass should own the work associated with a particular stage of rendering rather than becoming a second renderer by itself.

For example, a forward pass can consume prepared render work and issue the commands necessary for the forward scene stage. Future passes such as shadows can be implemented as separate stages while sharing the same lower-level Vulkan infrastructure.

## Presentation

Presentation-specific code is separated from general rendering work. Swapchain and frame-presentation concerns belong here rather than in gameplay-facing code.

## Direction

The renderer should expose progressively higher-level operations as it matures. Features such as materials, models, shadows, decals, and particles should normally be implemented against those abstractions rather than directly manipulating raw Vulkan objects from unrelated engine modules.
