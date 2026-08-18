# RenderSystemVK Layers

The Vulkan renderer is intentionally split so raw Vulkan object management does not become the API used by every graphics feature.

## High-level renderer

`CRenderSystemVK` is the renderer-facing implementation entry point in the `render` layer. It coordinates renderer initialization, frame work, resource preparation, and the lower-level Vulkan infrastructure required to execute rendering.

Feature code should trend toward this level or other high-level rendering abstractions rather than manually assembling Vulkan operations throughout the engine.

## Backend context

The `backend` directory currently contains `CVulkanContext` and `CVulkanAllocator`.

`CVulkanContext` centralizes foundational Vulkan state such as instance/device selection and device-facing context required by the rest of the renderer. `CVulkanAllocator` centralizes VMA-backed allocation concerns.

This makes the context/allocator infrastructure shared implementation rather than something every resource class must rediscover.

## Vulkan object wrappers

`vulkan/objects` contains dedicated wrappers for Vulkan resources and synchronization objects. The current tree includes wrappers for areas such as:

- buffers;
- command pools;
- descriptor pools and descriptor-set layouts;
- fences;
- graphics pipelines;
- and additional Vulkan resource types used by the renderer.

These wrappers are still low-level. Their purpose is RAII/lifetime and API encapsulation, not to become the interface used directly by gameplay systems.

## Why both layers matter

A wrapper around `VkBuffer` solves a different problem from a high-level GPU model resource. Likewise, a graphics-pipeline wrapper solves Vulkan pipeline lifetime/setup, while a renderer feature decides **which** pipeline is required and **when** it is used.

A useful mental model is:

```text
Game / Render Extraction
          │
          ▼
High-level RenderSystem / features
          │
          ▼
Render passes / GPU resources / pipelines
          │
          ▼
Vulkan wrappers
          │
          ▼
Vulkan + VMA
```

## Development rule

When implementing a graphics feature, first ask whether it can be expressed through an existing high-level renderer abstraction. Vulkan wrapper access should normally remain inside renderer implementation code. If every new feature needs to manipulate descriptor pools, command pools, fences, and raw pipeline setup itself, the high-level renderer boundary is too thin.

!!! warning
    RenderSystemVK is actively being refactored. This page documents the architectural distinction between layers; individual wrapper names and ownership details may still change.
