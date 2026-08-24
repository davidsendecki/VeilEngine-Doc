# Presentation & Resize

`CVkPresentationResources` owns and coordinates the complete window-presentation bundle. Resize and VSync notifications update requested state; creation and recreation occur later at a safe frame boundary.

## Owned resources

The presentation bundle owns:

- `CVkSwapchain` and its color image views;
- one render-finished semaphore per swapchain image;
- the committed layout tracked for each swapchain image;
- a depth image shared by the current presentation bundle;
- the selected depth format and committed depth-image layout;
- requested physical-pixel extent and VSync policy.

`CVkFrameContext` separately owns the image-available semaphore used to acquire an image. The render-finished semaphore belongs to the acquired swapchain-image index because presentation waits on completion of work targeting that specific image.

## Deferred recreation

`OnWindowPixelSizeChanged()` and `SetVSync()` do not immediately destroy the swapchain. They update requested state and mark recreation as pending.

At frame start, `EnsureReady()` returns:

| Status | Meaning |
| --- | --- |
| `Ready` | Existing resources are usable |
| `Recreated` | A new presentation bundle was created successfully |
| `Deferred` | Creation is postponed because the requested extent is unusable |
| `Failed` | Creation or recreation failed |

A zero width or height commonly means the window is minimized. Rendering is suspended without treating this temporary state as a renderer failure.

## VSync policy

The renderer starts with VSync disabled. Enabling VSync requests FIFO presentation. When VSync is disabled, swapchain creation selects the best supported low-latency presentation mode according to the current swapchain policy.

Changing VSync schedules the same safe recreation path used for size changes.

## Dynamic rendering

`BeginRendering()`:

1. transitions the acquired swapchain image from its tracked layout to color-attachment layout;
2. transitions the shared depth image to depth-attachment layout when required;
3. begins dynamic rendering with the acquired color view and shared depth view.

`EndRendering()` ends dynamic rendering and records the transition of the color image to `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`.

No traditional Vulkan render pass or framebuffer object is required for this path.

## Layout tracking

Recording a transition does not mean the GPU has accepted it. `MarkSubmitted()` commits the new CPU-side layout state only after the command buffer was submitted successfully.

If submission fails, recovery must not claim that recorded-but-unsubmitted transitions occurred.

## Pipeline compatibility

Presentation recreation reports whether the color or depth format changed. RenderSystemVK recreates pass pipelines only when their dynamic-rendering attachment formats are no longer compatible.

The requested extent may change without changing either format; that recreation does not require new graphics pipelines.

## Acquisition and presentation results

- `VK_ERROR_OUT_OF_DATE_KHR` requests recreation and skips or fails the affected presentation operation safely.
- `VK_SUBOPTIMAL_KHR` allows completion while requesting recreation for a later frame.
- Other acquisition or presentation errors are reported as failures.

Code outside RenderSystemVK should report window and VSync changes through `SRenderSysAPI`; it must not rebuild presentation objects directly.
