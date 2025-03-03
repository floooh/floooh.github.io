---
layout: post
title: The sokol-gfx compute update
---

In the next couple of days I will merge initial compute shader support
for sokol_gfx.h. The only breaking change is that the runtime feature flag
`sg_features.storage_buffer` has been renamed to `sg_features.compute`
(this is because the same backends that supported storage buffers before
now also support compute shaders).

Compute shader support is available on the following platform/backend combos:

- macOS and iOS with Metal
- Windows with D3D11 and GL
- Linux with GL
- Web with WebGPU

...this means compute shaders are not available on:

- macOS with GL
- iOS with GLES3
- Web with WebGL2
- Android with GLES3

The initial compute shader support comes with a couple of restricitions
which will most likely be lifted in later updates (in about that order):

- storage buffers cannot be bound as vertex- or index-buffers
- no storage textures, e.g. compute shaders can only write to storage buffers
- there's no way to read data from GPU resources back to the CPU side

This means that compute shaders are currently mostly useful for replacing
dynamic- and streaming-buffer update scenarios, where dynamic render
data is computed on the CPU and uploaded to buffers via `sg_update_buffer()`.

## API and Behaviour Changes

A quick rundown of API additions and changed behaviour:

- When initializing sokol-gfx via `sg_setup()`, a new config item
  `sg_desc.max_dispatch_calls_per_pass` has been added (default: 1024).
  Currently this is used only to preallocate an internal array which keeps
  track of modified storage buffers in a 'compute pass'.

- It's now allowed to create an immutable buffer without providing data,
  in this case the buffer content will be cleared to zero (but note that
  creating a dynamic or streaming buffer without data still leaves the
  buffer content in an 'undefined state'.

- The enum `sg_shader_stage` has new item `SG_SHADERSTAGE_COMPUTE`.

- When creating a shader via `sg_make_shader`, the `sg_shader_desc` struct
  has the following additional fields (which you can ignore when using the `sokol-shdc`
  shader cross-compiler):
    - An nested struct `compute_func` (similar to existing `vertex_func` and `fragment_func`)
      to pass a compute shader function as backend-specific source code
      or bytecode blob
    - A Metal-specific `mtl_threads_per_threadgroup` nested struct which
      communicates the 'workgroup size' extracted from shader code to the Metal API
    - The `readonly` boolean in the storage buffer bindslot declaration is now
      allowed to be false, but only in compute shaders. The readonly flag is
      now used by sokol-gfx for 'resource hazard tracking' in some backend APIs.
    - A new HLSL/D3D11 specific item `uint8_t register_u_n` has been added to
      the storage buffer bindslot declaration, this is used to communicate the
      HLSL bindslot for writable storage buffer bindings (which are bound as D3D11
      'unordered access views', while readonly storage buffers continue to be
      bound as 'shader resource views').

- Pipeline objects can now be either a 'render pipeline' or a 'compute pipeline'.
  To create a compute pipeline, set the new `sg_pipeline_desc.compute` flag to true
  and pass a with a valid `sg_shader_desc.compute_func`:

    ```c
    sg_pipeline pip = sg_make_shader(&(sg_pipeline_desc){
        .compute = true,
        .shader = compute_shader,
    });
    ```

- Passes now also come in two flavours: 'render passes' and 'compute passes'.
  A compute pass doesn't take a pass-action, attachments object or
  swapchain definition as inputs, instead `sg_begin_pass()` is called like
  this to start a compute pass:

    ```c
    sg_begin_pass(&(sg_pass){ .compute = true });
    ```

  The following restrictions exist inside a compute pass:
    - the functions `sg_apply_viewport[f]()`, `sg_apply_scissor_rect[f]()` and
      `sg_draw()` must not be called
    - `sg_apply_pipeline()` must be called with a compute pipeline object
    - `sg_apply_bindings()` cannot bind vertex- or index-buffers

  These restrictions are enforced by the sokol-gfx validation layer.

- A new function `sg_dispatch()` has been added to kick off a 'compute workload'.
  This function must only be called inside a compute pass:

    ```c
    sg_dispatch(int num_groups_x, int num_groups_y, int num_groups_z);
    ```
