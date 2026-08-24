# Frame Lifecycle

`CRenderSystemVK::RenderFrame()` coordinates presentation readiness, resource preparation, frame synchronization, pass preparation, command recording, submission, and presentation.

## Frames in flight

The renderer owns exactly two `CVkFrameContext` objects. Each frame context owns:

- an independent command pool;
- one reusable primary command buffer;
- an image-available semaphore used by swapchain acquisition;
- a fence tracking completion of the frame's queue submission;
- a stable frame index used to select pass-local per-frame data.

Before a frame slot is reused, `WaitForReuse()` waits for its previous submission. Per-frame buffers and descriptors can then be updated without overwriting data still consumed by the GPU from the other slot.

## Frame sequence

One successful frame follows this order:

1. Clear the previous render queue.
2. Validate `SRenderView` and `SRenderWorld`.
3. Return without rendering when no usable drawable extent is requested.
4. Defensively ensure referenced model and sky resources exist.
5. Ask `CVkPresentationResources` to create or recreate its bundle when required.
6. Ensure all pass pipelines exist and match the current color/depth formats.
7. Select the current `CVkFrameContext` and wait for it to become reusable.
8. Acquire a swapchain image with the frame's image-available semaphore.
9. Build and sort the transient render queue.
10. Prepare sky and forward-pass data for the selected frame slot.
11. Reset the frame command pool and begin primary-command-buffer recording.
12. Transition attachments and begin dynamic rendering.
13. Record `CSkyPass`, then `ForwardPass`.
14. End dynamic rendering and transition the swapchain image for presentation.
15. End and submit the command buffer, signaling the acquired image's render-finished semaphore.
16. Commit CPU-side attachment layout tracking after successful submission.
17. Advance the frame index and present the acquired image.

The render queue is transient CPU data. It references manager-owned GPU mesh handles and is cleared after its commands have been recorded.

## Render queue construction

For every `SModelRenderInstance`, RenderSystemVK resolves the model asset handle to an `SGPUStaticMeshHandle`. An unresolved or invalid resource fails queue construction rather than storing an unsafe draw request.

The queue stores:

- the resolved GPU mesh handle;
- the instance's world transform;
- optional render flags.

`RenderQueue::Sort()` defines the current ordering. Its exact strategy remains an implementation detail so it can later account for pipelines, materials, transparency, or other render state.

## Pass preparation and recording

Preparation performs CPU writes only after the frame slot is reusable:

```text
BuildRenderQueue
      │
      ├─ CSkyPass::PrepareFrame
      └─ ForwardPass::PrepareFrame
                 │
                 ▼
        begin dynamic rendering
                 │
                 ├─ CSkyPass::Record
                 └─ ForwardPass::Record
```

`ForwardPass::PrepareFrame()` writes aligned per-draw blocks into the selected frame's persistently mapped uniform buffer. `Record()` later binds the same frame's descriptor set with the matching dynamic offset.

## Recoverable presentation results

`VK_ERROR_OUT_OF_DATE_KHR` during acquisition schedules presentation recreation and skips the frame. `VK_SUBOPTIMAL_KHR` allows the current operation to complete but schedules recreation afterward.

## Frame recovery

Failures after acquisition may leave synchronization state unsuitable for immediate reuse. `CVkFrameContext::Recover()` rebuilds the affected synchronization state, distinguishing failures that occurred before `vkQueueSubmit2` from failures reported after submission was attempted.

Recovery also requests presentation-resource recreation. A frame context must not be reused if recovery itself fails.
