---
layout: post
title: "The new sokol-gfx WebGPU backend"
---

In a couple of days I will merge the new WebGPU backend in sokol_gfx.h, and instead of an oversized
changelog entry I guess it's time for a new blog post instead (the changelog entry will have all the
technical details too, but here I want to go a bit beyond that and also talk about the design decisions,
what went well and didn't and what to expect in the future).

## WebGPU in a Nutshell

From the perspective of sokol-gfx, WebGPU is a fairly straightforward API:

- a `WGPUDevice` object which creates other WebGPU objects
- `WGPUQueue`, `WGPUCommandBuffer`, `WGPUCommandEncoder` and `WGPURenderPassEncoder`
  objects for sending rendering commands to WebGPU
- `WGPUBuffer` objects to hold vertex-, index- and uniform-data
- `WGPTexture` objects to hold pixel data, and `WGPUTextureView` objects to
  describe subsections of that data
- `WGPUShaderModule` objects which encapsulate shader code
- `WGPUBindGroupLayout` and `WGPUPipelineLayout` objects which together
  describe the shader resource binding interface
- baked `WGPUBindGroup` objects for communicating uniform-buffer-, texture-
  and sampler-bindings to shaders
- `WGPUSampler` objects for describing how pixel data is sampled in shaders
- `WGPURenderPipelineState` objects which bake granular render states into a
  single immutable object

...and that's about it. Currently sokol-gfx doesn't use the following WebGPU
feature areas:

- storage buffers and textures
- compute passes
- render bundles
- occlusion and timestamp queries

WebGPU will slowly replace WebGL2 as the 'reference platform' to define
the sokol feature set. However, only for features that can also be implemented
without an emulation layer on top of D3D11, Metal and desktop GL.

Storage resource and compute passes are definitely at the top of the list,
while the idea of render bundles will most likely never make it.

## WebGPU in the Sokol Ecosystem

As with other backends, **sokol_gfx.h** expects that the `WGPUDevice` and
swapchain resources are created and managed externally. sokol-gfx
only depends on [<webgpu/webgpu.h>](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h).

Initially, **sokol_app.h** will only support setting up a WebGPU device
and swapchain for the Emscripten backend, as alternative to WebGL2.
This means that another 'window system glue' library (like GLFW) must be used
when sokol-gfx is used together with a native WebGPU library like
[Google's Dawn](https://dawn.googlesource.com/dawn).

The **sokol-shdc** shader compiler gains support for translating Vulkan-style
GLSL via SPIRV into WGSL with the help of [Google's Tint](https://dawn.googlesource.com/tint) library.

## WebGPU Backend Caveats

A quick rundown of how the 'tricky bits' are handled in the sokol-gfx WebGPU backend:

### Uniform Updates

sokol-gfx doesn't expose uniform buffers to the API user, instead uniform data is
considered to be 'frame-transient', all uniform data that's needed for rendering a frame
needs to be written in that frame. This usually makes sense because uniform data is often
highly dynamic and unpredictable (otherwise it probably shouldn't be uniform data in the
first place).

The WebGPU backend creates a single uniform buffer which must be big enough to hold
all uniform data for one worst-case frame (the size is tweakable at sokol-gfx
initialization). Due to some rather unfortunate design decisions in the WebGPU
Javascript API when used from WASM, the uniform data coming in via `sg_apply_uniform()`
calls is copied into an intermediate memory buffer in the WASM heap, and written
into the WebGPU uniform buffer with a single `wgpuQueueWriteBuffer()` call. This
currently seems to be the best compromise between simplicity and performance when using
WebGPU from WASM, but frankly, all options suck. The main problem is that a
Javascript ArrayBuffer object cannot be mapped into the WASM heap. So all data
transfers between WASM and WebGPU involve a copy operation between the WASM heap
and another JS ArrayBuffer object. This is an area where I'll keep experimenting
though by bypassing the Emscripten WebGPU shim, and instead write a handful of
my own specialized JS shim functions.

The uniform updating code actually went through several rewrites. The first version
used a "mapped-buffer conveyor belt", this was before the `wgpuQueueWriteBuffer()`
convenience function was added to the WebGPU API, which basically does the same
thing.

The next rewrite directly called `wgpuQueueWriteBuffer()` inside `sg_apply_uniforms()`,
but that showed up fairly prominently in profiling sessions, moving to an intermediate
copy and then doing a single big write-buffer operation at the end was actually better.

There's one icky high frequency call remaining though. `sg_apply_uniforms()`
needs to record a current uniform buffer offset for the next draw call. In
WebGPU the only way to do this is to call `wgpuRenderPassEncoderSetBindGroup()`
which is an incredibly 'fat' call for just updating a single buffer offset,
since it take a BindGroup object, and an array of offsets for **all** bindings
in the bindgroup, but only a single buffer offset has actually changed.

For comparison, Metal has special setBufferOffset calls exactly for this
purpose (to update a single buffer offset for an already bound buffer).

And currently this part causes the most grief, because this SetBindGroup() call
has more CPU overhead than WebGL2's glUniform4fv() call (which is used in in the
sokol-gfx WebGL2 backend for writing per-draw uniform data.

This produces the paradoxical situation that draw-call-intensive code (which
also always means that some uniform data is updated between draw calls,
otherwise there wouldn't be separate draw calls in the first place) - is
*faster* in WebGL2 than WebGPU (depending a lot on the platform and WebGPU
backend API, the biggest performance difference is on Windows with an NVIDIA
graphics card).

Of course this isn't set in stone (I hope at least!), WebGPU is still in the
"make it work" and not yet in the "make it fast" phase, and I really hope CPU
overhead for the SetBindGroup() call with dynamic offsets can be drastically
reduced so that it is at least in the same ballpark as the WebGL2
glUniform4fv() call, but even then it currently looks like WebGPU will require
just as many 'batching tricks' as WebGL2 (at least though WebGPU offers more
flexibility for batching), but don't expect that dumb WebGPU rendering code
automatically runs circles around equally dumb WebGL2 code, you'll still need
to reduce calls into the WebGPU API as much as possible.

This is a bit of a bummer to be honest, because for instance on macOS and iOS, just
porting dumb rendering code from OpenGL to Metal provided a dramatic performance boost
'for free' just because the internal API CPU overhead is a lot lower on Metal.

Funny enough I was never a fan of WebGPU's baked BindGroup objects, and it's
exactly the one area in the whole API which now causes the most trouble :D (I
didn't expect performance problems though, but instead that I think that
resource bindings should also be 'frame-transient' instead of being baked into
immutable objects - baked objects *should* actually be faster than transient
data, yet here we are with WebGL2 outperforming WebGPU).

Anyway, enough ranting :)


### Shader Interface Reflection Info

### Resource Bindings







## sokol-gfx to WebGPU function mapping

Here's what's happening under the hood in the sokol-gfx WebGPU backend:

- **sg_setup()**:
    - Creates one `WGPUBuffer` with usage `Uniform|CopyDst` big enough to receive all uniform
      updates for one frame
    - ...plus a `WGPUBindGroup` object to bind the uniform buffer to the vertex- and
      fragment-shader stage. The BindGroup will be bound to the reserved `@group` slot 0,
      with the first four `@binding` slots assigned to the vertex stage, and the following
      four `@binding` slots assigned to the fragment stage, for four different uniform update
      'frequencies' per stage.
    - an 'empty' `WGPUBindGroup` is created, for pipeline object that don't expect any texture and
      sampler bindings
    - an initial `WGPUCommandEncoder` will be created for the first frame



Mapping of sokol-gfx concepts to WebGPU is fairly straightforward, this is not surprising because both sokol_gfx.h and WebGPU draw a lot of inspiration from Metal
- which also means that the parts where WebGPU diverges from Metal and instead uses
ideas Vulkan, the mapping is more complex.

Currently the sokol-gfx feature set is still 'dominated' by WebGL2, which means
that sokol-gfx currently only implements a subset of WebGPU features. The general direction will
be to let WebGL2 no longer dictate the sokol_gfx.h feature set, but instead focus on the
cross-section between Metal, D3D11, GL4.1 and WebGPU. But this is still in the future.

This is how sokol-gfx maps to WebGPU right now:


- **sg_make_buffer()**:
    - Creates one `WGPUBuffer`, if necessary, rounds up the buffer size to
      the next multiple of 4.
    - If this is an immutable buffer, the WebGPU buffer will be created as
      'initially mapped' and the initial content will be copied into the buffer
      via `wgpuBufferGetMappedRange()` + `memcpy()` + `wgpuBufferUnmap()` (this is
      actually fairly inefficient on WASM because it involves redundant heap
      allocation and data copies in the Javascript shim, this will probably be
      optimized in a followup update)

- **sg_make_image()**:
    - Creates one `WGPUTexture` object and one `WGPUTextureView` object for accessing
      the texture from a shader stage.
    - If this is an immutable texture, the initial data will be copied into the
      texture with a series of `wgpuQueueWriteTexture()` calls

- **sg_make_sampler()**:
    - creates one `WGPUSampler` object and that's it.

- **sg_make_shader()**:
    - creates two `WGPUShaderModule` objects from the WGSL vertex- and fragment-shader
      source code provided in the `sg_shader_desc` struct
    - creates one `WGPUBindGroupLayout` from the shader interface reflection info provided
      in `sg_shader_desc`, texture/sampler bindings go into `@group` slot 1 and at predefined
      `@binding` slots:
        - vertex shader textures: start at `@binding(0)`
        - vertex shader smaplers: start at `@binding(16)`
        - fragment shader textures: start at `@binding(32)`
        - fragment shader samplers: start at `@binding(48)`

- **sg_make_pipeline()**
    - creates one `WGPUPipelineLayout` object which merges the global uniform buffer
      binding with the pipeline shader's texture/sampler bindings
    - creates one `WGPURenderPipeline` object

- **sg_make_pass()**
    - creates one `WGPUTextureView` object for each pass attachment (color-, resolve- and
      depth-stencil-attachments)

- **sg_begin_pass()**
    - FIXME

- **sg_apply_viewport()**
    - FIXME

- **sg_apply_scissor_rect()**
    - FIXME

- **sg_apply_pipeline()**
    - FIXME

- **sg_apply_bindings()**
    - FIXME

- **sg_apply_uniforms()**
    - FIXME

- **sg_wgpu_draw()**
    - FIXME

- **sg_end_pass()**
    - FIXME

- **sg_commit()**
    - FIXME

- **sg_update_buffer() / sg_append_buffer()**
    - calls `wgpuQueueWriteBuffer` once or twice (twice if the data size isn't
      a multiple of 4, in that case the dangling 1..3 bytes need to be copied
      separately)

- **sg_update_image()**
    - copies the data as a series of `wgpuQueueWriteTexture()` calls (same code
      that's used for populating immutable textures in `sg_make_image()`)

## Missing Features and Caveats

- 1010102 vertex format
- texture formats ...
- viewport clipping
- ...?