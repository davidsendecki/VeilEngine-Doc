# Render Frame & Presentation

`CRenderSystemVK` is the root owner of the current Vulkan renderer. It coordinates the Vulkan context and allocator, transfer infrastructure, swapchain, per-frame synchronization/resources, depth target, GPU model resources, and model renderer.

## Frame inputs

The engine calls `RenderFrame` with two client-owned snapshots:

- `SRenderView` — camera/view state;
- `SRenderWorld` — extracted renderable world state.

The renderer therefore consumes prepared rendering data rather than reaching back into the gameplay ECS.

## Frames in flight

The current renderer maintains exactly two independently writable frame-resource sets. The frame index selects which set is used for the current submission, allowing CPU preparation to proceed without immediately reusing resources still owned by an earlier GPU submission.

## Presentation resources

`CVulkanSwapchain` owns swapchain-specific presentation behavior. `CRenderSystemVK` additionally tracks one render-finished semaphore per swapchain image and the known layout of each swapchain image.

The renderer also owns a shared depth image and tracks its current layout.

## Resize and VSync

Window pixel-size changes do not immediately tear down the swapchain from an arbitrary callback. The new physical extent is recorded and recreation is deferred to a safe rendering boundary.

Changing VSync similarly changes presentation policy and schedules swapchain recreation.

A zero drawable extent suspends normal rendering, which is important for minimized windows.

## Swapchain recreation

Swapchain recreation is a renderer-level operation because presentation resources are interconnected with frame rendering. Code outside the renderer should report window/presentation changes rather than manually rebuilding Vulkan presentation objects.

## Shutdown

Shutdown waits for the device and releases resources in reverse dependency order. Foundation objects such as transfer infrastructure, allocator, and Vulkan context are intentionally destroyed after resources that borrow them.
