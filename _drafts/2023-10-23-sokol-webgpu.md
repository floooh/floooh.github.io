---
layout: post
title: "The new sokol-gfx WebGPU backend"
---

In a couple of days I will merge the new WebGPU backend in sokol_gfx.h, and instead of an oversized
changelog entry I guess it's time for a new blog post instead (the changelog entry will have all the
technical details too, but here I want to go a bit beyond that and also talk about the design decisions,
what went well and didn't and what to expect in the future).

## 10000ft View

The changes in the next update go beyond just sokol_gfx.h. All of the following
points will be explained in more details later.

- **sokol_gfx.h**: a new WebGPU backend activated with the define `SOKOL_WGPU`, this
  depends only on `webgpu/webgpu.h` (so it works both on the web and with native
  WebGPU implementations)
- **sokol_app.h**: the Emscripten backend can now setup a WebGPU device and swapchain,
  native backends currently don't support WebGPU, also on the web there is no
  fallback from WebGPU to WebGL, instead two separate WASM blobs need to be compiled
  if a fallback solution is desired
- **sokol-shdc** (the sokol shader compiler) gets a wgsl output option, implemented via
  Tint (basically Google's SPIRVCross), and two new tags to hint the shader reflection
  code generation about intended texture and sampler usage
- **sokol-samples** a new set of low-level WebGPU samples which can be built for the
  web or Google's libdawn, and a handful new cross-platform / cross-backend samples.
  Also a new sample webpage is hostedhere: https://floooh.github.io/sokol-webgpu/

## How sokol-gfx maps to WebGPU

Mapping of sokol-gfx concepts to WebGPU is fairly straightforward, this is not surprising because both sokol_gfx.h and WebGPU draw a lot of inspiration from Metal
- which also means that the parts where WebGPU diverges from Metal and instead uses
ideas Vulkan, the mapping is more complex.

Currently the sokol-gfx feature set is still 'dominated' by WebGL2, which means
that sokol-gfx currently only implements a subset of WebGPU features. The general direction will
be to let WebGL2 no longer dictate the sokol_gfx.h feature set, but instead focus on the
cross-section between Metal, D3D11, GL4.1 and WebGPU. But this is still in the future.

This is how sokol-gfx maps to WebGPU right now:

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