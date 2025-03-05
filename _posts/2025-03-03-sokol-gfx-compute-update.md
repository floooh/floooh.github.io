---
layout: post
title: The sokol-gfx compute shader update
---

> NOTE: links to the WebGPU live samples will be broken until after the merge

In the next couple of days I will merge initial compute shader support
for sokol_gfx.h (and sokol-shdc). The update is surprisingly 'low-profile' in terms
of API changes, the only breaking change is that the runtime feature flag
`sg_features.storage_buffer` has been renamed to `sg_features.compute`
(this is because the same backends that supported storage buffers before
now also support compute shaders).

## Availability and Restrictions

Compute shader support is available on the following platform/backend combos:

- macOS and iOS with Metal
- Windows with D3D11 and GL
- Linux with GL
- Web with WebGPU

...which means that compute shaders are not available on:

- macOS with GL
- iOS with GLES3
- Web with WebGL2
- Android with GLES3

The initial compute shader support comes with a couple of restricitions
which will most likely be lifted in later updates (in about that order):

- storage buffers cannot be bound as vertex- or index-buffers
- no storage textures, e.g. compute shaders can only write buffer data but not texture data
- there's no way to read data from GPU resources back to the CPU side (or
  copy data between GPU resources)

Right now compute shaders are mostly useful for replacing
dynamic- and streaming-buffer update scenarios, where dynamic render
data is computed on the CPU and uploaded to buffers via `sg_update_buffer()`.

## New compute shader samples

To get an idea how compute shaders work in sokol-gfx, it's best to read the
new sample code:

- [C code](https://github.com/floooh/sokol-samples/blob/sgcompute/sapp/instancing-compute-sapp.c)
- [GLSL code](https://github.com/floooh/sokol-samples/blob/sgcompute/sapp/instancing-compute-sapp.glsl)
- [WebGPU demo](https://floooh.github.io/sokol-webgpu/instancing-compute-sapp.html)

This is an evolution of the [instancing-sapp](https://floooh.github.io/sokol-webgpu/instancing-sapp-ui.html)
sample, and moves all particle computations into compute shaders.

The other compute shader sample is a straight port of the [WebGPU compute boids sample](https://webgpu.github.io/webgpu-samples/?sample=computeBoids) to
sokol-gfx:

- [C code](https://github.com/floooh/sokol-samples/blob/sgcompute/sapp/computeboids-sapp.c)
- [GLSL code](https://github.com/floooh/sokol-samples/blob/sgcompute/sapp/computeboids-sapp.glsl)
- [WebGPU demo](https://floooh.github.io/sokol-webgpu/computeboids-sapp.html)

Those two samples use 'cross-backend' GLSL shader code compiled to the underlying
shading languages via [sokol-shdc](https://github.com/floooh/sokol-tools/).

For authoring compute shaders with sokol-shdc it might make sense to read up
on [GLSL compute shaders in the GL Wiki](https://www.khronos.org/opengl/wiki/Compute_Shader) -
note though that not all features have been properly tested yet (like sampling
textures in compute shaders, or accessing shared memory).

For using sokol-gfx compute shaders without sokol-shdc, check out the following
backend specific versions of the `instancing-compute` sample:

- D3D11: [instancing-compute-d3d11.c](https://github.com/floooh/sokol-samples/blob/sgcompute/d3d11/instancing-compute-d3d11.c)
- Metal: [instancing-compute-metal.c](https://github.com/floooh/sokol-samples/blob/sgcompute/metal/instancing-compute-metal.c)
- WebGPU: [instancing-compute-wgpu.c](https://github.com/floooh/sokol-samples/blob/sgcompute/wgpu/instancing-compute-wgpu.c)
- GL4.3: [instancing-compute-glfw.c](https://github.com/floooh/sokol-samples/blob/sgcompute/glfw/instancing-compute-glfw.c)

Also check out the updated documentation of [sokol-shdc](https://github.com/floooh/sokol-tools/blob/sgcompute/docs/sokol-shdc.md),
and the new documentation comment section on compute shaders in the sokol_gfx.h
header (search for: `ON COMPUTE PASSES` and re-read the updated section `ON SHADER CREATION`).

## Shader Authoring Changes

The sokol-gfx update comes with a matching sokol-shdc update for
authoring compute shaders.

A new tag `@cs [name]` (similar to the existing `@vs [name]` and `@fs [name]`)
is used to identify a compute shader snippet, e.g. everything inside `@cs / @end`
will be compiled as a [GLSL compute shader](https://www.khronos.org/opengl/wiki/Compute_Shader).

NOTE that the distinction between readonly and read/write storage buffer
bindings is important, e.g.:

```glsl
layout(binding=0) readonly buffer cs_ssbo_in { particle prt_in[]; };
layout(binding=1) buffer cs_ssbo_out { particle prt_out[]; };
```

If your compute shader only reads (but doesn't write) storage buffer content,
its binding declaration should be marked as `readonly`. This information will
be extracted by sokol-shdc and used by sokol-gfx for hazard-tracking
needed in some 3D-APIs.

The other notable shader specialty is the 'workgroup size', which in
GLSL is defined as:

```glsl
layout(local_size_x=X, local_size_y=Y, local_size_z=Z) in;
```

...if you're used to HLSL, this is the same as `[numthreads(X,Y,Z)]`, or in WGSL
`@workgroup_size(X,Y,Z)`. On Metal this is called `threadsPerThreadGroup` and
is **not** defined in the shader code, but on the CPU side when issuing a dispatch
call (this is another case where sokol-shdc comes in handy, since it extracts
the workgroup size from the GLSL shader and passes it into sokol-gfx as
`sg_shader_desc.mtl_threads_per_threadgroup`).

Other then that you mainly need to be aware that your compute shader code must
be thread safe because compute shaders have random write access into storage buffers
and the GPU is spawning many invocations of your shader running in parallel.

## On the CPU side

The `sg_setup()` call gets a new config item `sg_desc.max_dispatch_calls_per_pass`
(default: 1024). This is used to allocate an internal array to keep track of
written storage buffers in a compute pass for hazard tracking purposes.

There's a minor change when creating buffers: It's now allowed to create
immutable buffers without initial content, and such buffers will be
zero-initialized (note though that dynamic- and streaming-buffers may
still have undefined buffer content after creation). This is useful
when using a compute shader to write the initial buffer content instead
of generating the data on the CPU side.

Shaders, pipelines and passes now come in two runtime flavours: 'render' vs 'compute',
where the 'render flavours' are fully compatible with existing code.

For shaders, nothing changes either when using sokol-shdc for shader authoring.
In that case you just write a compute shader and sokol-shdc will code-generate
a matching `sg_shader_desc` struct which can be plugged directly into the
`sg_make_shader()` call.

A compute pipeline is a regular pipeline object without any render state,
but with a compute shader attached:

```c
sg_pipeline pip = sg_make_shader(&(sg_shader_desc){
  .compute = true,
  .shader = a_compute_shader,
});
```

Finally, kicking off 'compute workloads' happens with a new function `sg_dispatch()`
inside 'compute passes':

```c
sg_begin_pass(&(sg_pass){ .compute = true });
sg_apply_pipeline(pip);
sg_apply_bindings(...);
sg_apply_uniforms(...);
sg_dispatch(x, y, z);
sg_end_pass();
```

The `sg_dispatch()` call takes the number of 'workgroups' as arguments
(same convention as GL, D3D11 and WebGPU, but different from Metal's `dispatchThreads` method).

Compute- vs render-passes now impose a couple of restrictions (checked
by the validation layer):

- the following functions must only be called in render passes:
  - `sg_apply_viewport[f]()`
  - `sg_apply_scissor_rect[f]()`
  - `sg_draw()`
- `sg_dispatch()` must only be called in a compute pass
- `sg_apply_bindings()` in a compute pass must not attempt to bind vertex- or index-buffers
- the `sg_apply_pipeline()` pipeline type must match the pass type

## When not using sokol-shdc

If you don't use sokol-shdc for shader authoring you'll need to populate the
all-important `sg_shader_desc` struct passed into `sg_make_shader()` yourself
with information that matches your shader code:

- An nested struct `compute_func` has been added (similar to existing
  `vertex_func` and `fragment_func`) to pass a compute shader function as
  backend-specific source code or bytecode blob
- A Metal-specific `mtl_threads_per_threadgroup` nested struct which
  defines the 'workgroup size' to the Metal API (this is in `sg_shader_desc`
  because those values are normally extracted from shader code via reflection)
- The `readonly` boolean in the storage buffer bindslot declaration is now
  allowed to be false, but only in compute shaders. The readonly flag is
  now used by sokol-gfx as hint for 'resource hazard tracking' in some backend APIs.
- A new HLSL/D3D11 specific item `uint8_t register_u_n` has been added to
  the storage buffer bindslot declaration, this is used to communicate the
  HLSL bindslot for writable storage buffer bindings (which are bound as D3D11
  'unordered access views', while readonly storage buffers continue to be
  bound as 'shader resource views').

Also please carefully review the backend-specific compute shader samples
which directly pass backend-specific shader code into sokol-gfx:

- D3D11: [instancing-compute-d3d11.c](https://github.com/floooh/sokol-samples/blob/sgcompute/d3d11/instancing-compute-d3d11.c)
- Metal: [instancing-compute-metal.c](https://github.com/floooh/sokol-samples/blob/sgcompute/metal/instancing-compute-metal.c)
- WebGPU: [instancing-compute-wgpu.c](https://github.com/floooh/sokol-samples/blob/sgcompute/wgpu/instancing-compute-wgpu.c)
- GL4.3: [instancing-compute-glfw.c](https://github.com/floooh/sokol-samples/blob/sgcompute/glfw/instancing-compute-glfw.c)

## Under the hood

Most of the new code in sokol_gfx.h is just a straight-forward mapping from
sokol-gfx types and functions into backend 3D-API types and functions.

Only two details are worth mentioning:

- On Metal, and only on systems without unified memory, GPU-written
  managed storage buffers are 'synchronized' at the end of a compute
  pass inside `sg_end_pass()`. This synchronization basically updates the
  CPU-side shadow copy of the buffer with the new data that's been written
  by a compute shader. This requires keeping track of all read/write storage
  buffer bindings inside a compute pass (this is why the new `sg_desc.max_dispatch_calls_per_pass`
  config item is used for).
- On GL, `glMemoryBarrier()` calls are issued (at most once per `sg_apply_bindings()`
  call) when a storage buffer was previously bound as read/write (which sets
  an internal 'gpu_dirty' flag).

## What's next

...mainly patching remaining feature gaps in a couple of minor updates:

- allow storage buffers to be bound as vertex- and index-buffers
- introducing storage textures which can be written by compute shaders
- more 'feature coverage' by writing a handful more interesting compute samples

...and what will most likely a bigger update: figure out a proper
sub-API for `CPU => GPU`, `GPU => CPU` and `GPU => GPU` copies.
