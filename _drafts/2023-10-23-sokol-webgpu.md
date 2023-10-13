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

## The Tricky Parts

Most of the WebGPU backend is a straightforward mapping between C structs and
function calls, but some parts are more complex:

### Uniform Data Updates

sokol-gfx doesn't expose uniform buffers to the API user, instead uniform data is
considered to be 'frame-transient', all uniform data that's needed for rendering a frame
needs to be written in that frame by calling sg_apply_uniforms(). Each shader
stage offers 4 uniform data 'slots' to allow different update frequencies.

The WebGPU backend creates a single uniform buffer which must be big enough to hold
all uniform data for one worst-case frame (the size is tweakable at sokol-gfx
initialization). Apart from that an intermediate memory buffer of the same size is allocated
on the heap.

When new uniform data is coming in via `sg_apply_uniforms()`, the data is
appended to the intermediate memory buffer and the new offset is recorded into
the render pass encoder by calling `wgpuRenderPassSetBindGroup` with dynamic
buffer offsets. 8 offsets needs to be updated here even though only one
has changed.

At the end of the frame in `sg_commit()` a single big `wgpuQueueWriteBuffer` copies
the new uniform data from the intermediate memory buffer into the WebGPU
uniform buffer.

This uniform update mechanism works similar like in the native Metal backend, but
with two important differences: In the Metal backend, there is no intermediate
copy, the data is written directly into one of multiple uniform buffers which
are rotated through. And Metal offers special 'slim' functions to record only
a single buffer offset without any redundant data.

When trying to implement the same update mechanism in WebGPU and WASM, there
are multiple options, but all of them are not great:

- The first option is to implement a 'buffer conveyor belt', this basically
  rotates through multiple buffers, where the current write buffer is mapped
  for the duration of a frame. This was also the first version I implemented
  for an earlier version of the sokol-gfx WebGPU backend (before the
  write-buffer convenience function existed, which basically does the same
  thing). The problem with this approach is that a WebGPU buffer cannot be
  mapped directly into the WASM heap, instead the mapping happens to a
  separate Javascript ArrayBuffer object. This means in WASM it's not
  possible to read or write directly a mapped buffer via the WASM heap, and
  on top of that, the Emscripten WebGPU shim functions which wrap the
  buffer mapping need to do a temporary allocation in the WASM heap.

- Once the writeBuffer() convenient method was added to WebGPU the above
  solution didn't make much sense anymore because writeBuffer() essentially
  implements the same 'buffer conveyor belt' under the hood. So next version I
  tried is calling writeBuffer() directly from the `sg_apply_uniforms()`
  function (which is a very high frequency call). Unfortunately this added
  significant overhead, so I ditched that and that's when I arrived at the
  current solution using the intermediate memory buffer and a single big
  writeBuffer() per frame.

- Another option (which I'll have to try at some point) is to go back to the
  'uniform buffer conveyor belt', but write my own Javascript shim functions
  which wrap the buffer mapping and updating. This new shim function would be
  called at high frequency from `sg_apply_uniforms()`, and copy the original
  uniform data from the WASM heap into the separate ArrayBuffer object which
  represents the mapped uniform buffer, all on the Javascript side, and without
  going through an intermediate copy. This should be the best option but
  requires separate code paths for the WebGPU backend running in the browser
  and on native WebGPU libraries.

One problem that can't be fixed in sokol-gfx is that the WebGPU setBindGroup()
call has a surprisingly high CPU overhead. Long story short, updating uniform
buffer offsets via setBindGroup() is a lot slower currently than the WebGL2
function glUniform4fv() (which is used in the sokol-gfx GL backend to update
uniform data).

This means that micro-benchmarks which measure per-draw CPU overhead are currently
running slower in WebGPU than WebGL2.

Case in point: [DrawCallPerf sample for WebGL2](https://floooh.github.io/sokol-html5/drawcallperf-sapp.html)
versus [the same sample for WebGPU](https://floooh.github.io/sokol-webgpu/drawcallperf-sapp.html).

Hopefully these are just teething problems and can be fixed, because next to
the actual draw functions (which are fast enough), setBindGroup() is the next
most important high-frequency function which **must** have minimal CPU
overhead.

I have a lot of opinions about the BindGroup design decision, but I don't want
to derail this blog post into a rant, so let's move on :)

### Texture and Sampler Resource Bindings

The problems here are a bit similar to the uniform data update problem above, but with
a different solution. Again it's about BindGroups though :)

The common theme is that I consider resource bindings (specifically texture and
sampler bindings) also as frame-transient, and not as baked data. This collides
with WebGPU's philophy that resource bindings live in baked BindGroup objects.

(...ideally I would like something like a BindingsBuffer where a single buffer can
hold the resource bindings for an entire frame and I'm just recording offsets
into this BindingsBuffer before a drawcall - the luxus version of this would
be Metal-style argument buffers - but I disgress)

Back to sokol-gfx, here binding resources happens through a 'dynamic'
`sg_apply_bindings` call which simply takes arrays of textures and samplers
for the vertex- and fragment-shader stages. A 'dumb' mapping to WebGPU would
be to create an adhoc `WGPUBindGroup` object, apply that with setBindGroup()
and immediately release the object again (since the WebGPU C API is refcounted,
the BindGroup will then be released automatically when no longer needed by
the RenderPassEncoder). And that's what I initially implemented.

It works of course, but due to the surprisingly high CPU overhead of the
setBindGroup() call this wasn't an option (and besides, this is also not
exactly great for the Javascript garbage collector - I actually saw the typical
garbage collector spikes in the DrawCallPerf sample I linked above).

The current solution is caching and reusing BindGroup objects backed
by a very simple hashmap, the hashmap internals are exposed with a new
function `sg_query_frame_stats()` which allows to check for hashmap
slot collisions (because in this case, a BindGroup will be discarded and
another one created, and if this happens too frequently it's obviously
a bad thing - but should be fixable by increasing the number of slots
in the hash map at sokol-gfx initialization time).

### Shader Interface Reflection Info

WebGPU is both a very convenient API, but also extremely picky when
it comes to correctness. For a web API it's important that the
both the API specification and implementation are watertight and
catch all sorts of problems where native APIs allow a lot more slack.

The most gnarly of those issues is that texture/sampler combinations
must obey rules that are often underspecified in native APIs.

Long story short: WebGPU has a concept of 'texture-sample-type'
and 'sampler-binding-types' which must be defined upfront for
texture and sampler bindings when creating a BindGroupLayout object
(meaning basically that you need to know upfront what specific types
of textures and samplers you're going to use with a specific
render pipeline object).

A texture-sample-type can be:

- float
- unfilterable-float
- s(igned)int
- u(nsigned)int
- depth

...and sampler-binding-types can be:

- filtering
- non-filtering
- comparison

Some combinations of texture-sample-type and sampler-binding-types are illegal
and will be caught by WebGPU when creating a pipeline object by analyzing
and matching the provided shader code against the pipeline's BindGroupLayouts.

The problem for sokol-gfx is that this shader reflection information (what
textures and sampler combination are used by the shader) is internal to
WebGPU and cannot be queried by the API user. Sokol-gfx doesn't have the
concept of BindGroupLayouts, but it has something similar: when a shader
object is created, additional shader interface reflection info must be
provided for each shader stage:

- the number, size and (for GL) the internal layout of uniform blocks
- information about each texture that's used in the shader
- ...and information about each sampler

When the **sokol-shdc** offline shader compiler is used, all that information
is extracted from the shader, so usually the sokol-gfx API user won't need
to care about those dirty details. But there's a problem:

**sokol-shdc** cannot know just by looking at GLSL code whether a texture
will be of an `unfilterable-float` type. There's simply no keyword for that
in GLSL (and it is actually legal to bind either a filterable, or an unfilterable
texture to the same sampler, as long as the sampler is compatible).

I'm solving this with two new meta-tags in the sokol-shdc shader input to
provide those hints to the reflection code generation looking like this:

```glsl
@image_sample_type joint_tex unfilterable_float
uniform texture2D joint_tex;
@sampler_type smp nonfiltering
uniform sampler smp;
```

...looks a bit awkward - I know. Thankfully that's only need in some situations
(specifically when using float pixel formats), all other cases (depth, sint or
uint textures) are should be automatically detected by the **sokol-shdc**
shader reflection code.









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