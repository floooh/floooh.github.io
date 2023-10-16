---
layout: post
title: "The new sokol-gfx WebGPU backend"
---

In a couple of days I will merge the new WebGPU backend in sokol_gfx.h, and instead of an oversized
changelog entry I guess it's time for a new blog post instead (the changelog entry will have all the
technical details too, but here I want to go a bit beyond that and also talk about the design decisions,
what went well and didn't and what to expect in the future).

The sokol WebGPU backend samples are hosted here: [https://floooh.github.io/sokol-webgpu](https://floooh.github.io/sokol-webgpu).

However, the source-code links will point to outdated code until the WebGPU branch is merged into master.

## Table of Content

* TOC
{:toc}

## WebGPU in a Nutshell

From the perspective of sokol-gfx, WebGPU is a fairly straightforward API:

- a `WGPUDevice` object which creates other WebGPU objects
- `WGPUQueue`, `WGPUCommandBuffer` and `WGPUCommandEncoder` as general
  infrastructure for sending commands to the GPU
- `WGPURenderPassEncoder` to record commands for one render pass
  into a command encoder
- `WGPUBuffer` objects to hold vertex-, index- and uniform-data
- `WGPUTexture` objects to hold pixel data in a set of 'subimages', and
  `WGPUTextureView` objects to define a related group of such subimages.
- `WGPUSampler` objects for describing how pixel data is sampled in shaders
- `WGPUShaderModule` objects which compile WGSL shader code into an internal
  representation
- `WGPUBindGroupLayout` and `WGPUPipelineLayout` which together define
  an interface for how shader-resource-objects (uniform-buffers, texture-views
  and samplers) are communicated to shaders
- `WGPUBindGroup` objects for storing immutable shader-resource-object combinations
- `WGPURenderPipelineState` to group a vertex layout, shaders, a resource
  binding interface and granular render states into a single immutable
  state object

...and that's about it. Currently sokol-gfx doesn't use the following WebGPU
features:

- storage-buffers and -textures
- compute-passes
- render-bundles
- occlusion- and timestamp-queries

WebGPU will slowly replace WebGL2 as the 'reference platform' to define the
future sokol-gfx feature set (however, only for features that can also be
implemented without an emulation layer on top of D3D11, Metal and desktop GL).

Storage resources and compute passes are definitely at the top of the list,
while the idea of render bundles will most likely never make it into sokol-gfx.

(I'm really not a fan of render bundles, they look like a cheap cop-out to not
having to focus on optimizing the CPU overhead of high-frequency functions, and
since render bundles are essentially GL-style display lists (which were also a
bad idea that didn't stick) it will be hard to move them entirely onto the GPU
anyway, at best they can be played back inside the browser render process on
the CPU - but maybe there's a grand plan behind render bundles which I didn't
understand yet).

## WebGPU in the Sokol Ecosystem

As with other 3D backends, **sokol_gfx.h** expects that the device object and
swapchain resources are created and managed externally. Sokol-gfx
itself only depends on [<webgpu/webgpu.h>](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h).

Initially, **sokol_app.h** will only support WebGPU in the Emscripten backend
as alternative to WebGL2. This means that sokol_app.h can't be used together
with a native WebGPU implementation like Dawn or wgpu.rs, instead an alternative
window system glue library like GLFW must be used (currently you are
better off with the sokol-gfx backends for D3D11 and Metal anyway for native platforms, at
least on Windows and macOS).

The **sokol-shdc** shader compiler gains support for translating Vulkan-style
GLSL via SPIRV into WGSL with the help of [Google's Tint](https://dawn.googlesource.com/tint) library.

The custom 'annotated GLSL' accepted by sokol-shdc gained two new tags for hinting
reflection information about textures and samplers needed by WebGPU that
can't be inferred from the shader code. More on that later in this blog post.

## The Gnarly Parts

Most of the sokol-gfx WebGPU backend is a straightforward mapping
to WebGPU structs and functions, there are some notable exceptions though:

### Uniform Data Updates

Sokol-gfx doesn't expose the concept of uniform buffers to the API user,
instead uniform data is considered to be 'frame-transient', all uniform data
required for a frame must be written from scratch via sg_apply_uniforms() calls
interleaved with sg_draw() calls (this make senses because most uniform data
rarely remains unchanged between frames). Each sokol-gfx shader stage offers 4
uniform 'slots' which allows to supply uniform data snippets at different
update frequencies and map them to up to four uniform block structures per
shader stage in the shader code.

At startup, the sokol-gfx WebGPU backend creates a single uniform buffer which
must be big enough to hold all uniform data for one worst-case frame (including
a worst-case 256-byte alignment between uniform data snippets). Additionally,
an intermediate memory buffer of the same size is allocated on the heap.

When new uniform data is coming in via `sg_apply_uniforms()`, the data is
appended to the intermediate memory buffer and the offset of the new data is
recorded into the WebGPU render pass encoder by calling `wgpuRenderPassSetBindGroup`
with dynamic buffer offsets (where only one of 8 offsets has actually changed).

At the end of the frame in `sg_commit()`, a single big `wgpuQueueWriteBuffer` records
a copy operation from the intermediate WASM heap buffer into the WebGPU uniform
buffer.

This uniform update mechanism works similar to the native Metal backend, but
with two important differences: In the Metal backend, there is no intermediate
CPU memory buffer, the data is written directly into one of multiple uniform
buffers which are rotated each frame. For updating the uniform buffer offset,
Metal offers special 'shortcut' functions to record only a single buffer offset
for one buffer bind slot without having to rebind the buffer or updating unrelated offsets.

I went through several rewrites of the uniform update code, and the current
version most likely isn't the last one (it's good enough for the initial
implementation though, since it's the best compromise without having to
write different code paths for native platforms vs WebAssembly):

- The first version implemented a 'buffer conveyor belt' model, this basically
  rotates through multiple buffers, where the current frame's uniform buffer is
  mapped for the duration of a frame. This version is quite similar to the
  Metal backend. The problem when using WebGPU from WASM however is that a
  WebGPU buffer cannot be mapped directly into the WASM heap. The result of
  mapping a WebGPU buffer is a Javascript ArrayBuffer object, while the WASM
  heap is a separate ArrayBuffer object. WASM can only directly access its
  own WASM heap ArrayBuffer, but not separate buffers. This means that the Emscripten
  WebGPU Javascript shim needs to "emulate" buffer mapping via a temporary
  heap allocation and copying data between JS ArrayBuffer objects. Clearly
  this isn't exactly efficient (paradoxically a situation where pure Javascript
  WebGPU code will be faster than using WebGPU from WASM - there are workarounds
  though which I will be getting to in the last point).

- The next uniform-update rewrite called a new `wgpuQueueWriteBuffer` function
  once per sg_apply_uniform() call (this new function didn't exist yet when the
  first version of the uniform update code was implemented). This writeBuffer
  function essentially implements the same buffer-conveyor-belt inside the WebGPU
  implementation which I had implemented manually in the first version. The
  Javascript shim for the writeBuffer() call doesn't need to go through a
  temporary WASM heap allocation which is a pretty big plus.
  Unfortunately it turned out that the writeBuffer call overhead was still too
  much for such a high-frequency operation, doing writeBuffer at draw-call-frequency
  turned out to be a pretty bad idea.

- And thus the current version was born, which accumulates all uniform
  data snippets for an entire frame and then does a single big writeBuffer()
  call at the end of the frame in sg_commit().

- There's another option which I will try out at a later time (because that
  intermediate memory buffer really bothers me), but this method requires
  different code paths for WASM vs native platforms: This method would go back
  to a manually implemented buffer-conveyor-belt, but call out to a handful of
  my own specialized Javascript shim functions. At the start of a frame, a JS
  function would be called which obtains a mapped JS ArrayBuffer from the
  current frame's uniform buffer and that ArrayBuffer would be stored somewhere
  on the JS side until the end of the frame. During the frame the WebGPU
  backend version of sg_apply_uniforms() would call out into another Javascript
  shim function which directly copies the uniform data snippet from the WASM heap
  object into the mapped ArrayBuffer object. Finally at the end of the frame,
  a third Javascript shim function is called which unmaps the uniform
  ArrayBuffer. On native platforms, the regular WebGPU buffer mapping functions
  would be used instead of those specialized JS shim functions. Another advantage
  of that approach is that no copy-cost must be paid for the padding bytes
  between uniform data snippets (which can be substantial if the alignment
  is 256 bytes).

The next problem with WebGPU uniform buffer updates is unfixable on my side though:

Apart from copying the uniform data snippets into a uniform buffer,
sg_apply_uniforms() also needs to record the offset of new data in the uniform
buffer so that the next draw call knows where its uniform data starts in the
buffer.

In WebGPU this requires a setBindGroup() call with so-called 'dynamic buffer offsets'.

And currently, this setBindGroup() call has a surprisingly high CPU overhead
which makes the `sg_apply_uniforms()` call in the WebGPU backend significantly slower
than in the WebGL2 backend on some platforms. How big the difference is depends
a lot on the host platform, but as an example: on my (quite modest) Windows PC
(2.9 GHz i5 CPU and NVIDIA 2070) in Chrome with WebGL2 I can issue about 64k
uniform-update/draw-call pairs before frame rate starts to drop below 60Hz,
while on WebGPU it tops out at around 16k uniform-update/draw-call pairs (for
comparison: the native D3D11 backend goes up to 450k(!)
uniform-update/draw-call pairs on that PC before frame rate drops below
60hz).

On my M1 Mac the picture is actually quite different (NOTE that this uses a
120Hz framerate instead of 60Hz, to compare to the PC numbers you'll need to
double the macOS numbers!): WebGL2 is actually slightly slower than WebGPU here
(last time I looked at it that wasn't the case, so maybe some optimization work
is already happening?): 8.5k draws for WebGL2 vs 11k for WebGPU before a 120Hz
frame rate can no longer be sustained, for comparison, the same code compiled as
native and using the sokol-gfx Metal backend goes up to around 110k draws
before framerate drops below 120Hz.

The culprit is the setBindGroup() call which is somewhere between "about equal"
to "a lot slower" than the glUniform4fv() call that's used in the WebGL2
backend to update uniform data.

You can check for yourself in the following demo for
[WebGL2](https://floooh.github.io/sokol-html5/drawcallperf-sapp.html) and
[WebGPU](https://floooh.github.io/sokol-webgpu/drawcallperf-sapp.html).

It was definitely surprising to me that there are situations where WebGPU
can be drastically slower than WebGL2, and hopefully this is just a case
of "first make it work, then make it fast". But it looks like moving
from WebGL2 to WebGPU won't be such a no-brainer like moving from GLES3
to Metal on iOS, which came with a hefty performance increase without
having to change the frame rendering structure.

### Texture and Sampler Resource Bindings

Another BindGroup-related problem, what a surprise ;)

This time it's about the texture and sampler resource bindings.

Just as with uniform data, sokol-gfx considers shader resource bindings to be
frame-transient, e.g. they need to be written from scratch each frame (because
what else are shader resource bindings if not uniform data that's passed 'by
reference' instead of 'by value').

The motivation for this isn't quite as clear-cut as for uniform data though. In
games for instance, material systems can often stamp out all required material
instances upfront. But especially in non-game-applications, resource binding
combinations are often unpredictable, which can lead to combinatorial
explosions if resource bindings have to be baked upfront into immutable objects.

Resource binding in sokol-gfx happens through the `sg_apply_bindings()` call
which takes arrays of texture and sampler handles for each shader stage.

In WebGPU, those textures and samplers must be baked into a new BindGroup object.
This means that a dumb sokol-gfx backend would simply create a new BindGroup object
inside `sg_apply_bindings()`, call setBindGroup() on the render pass encoder,
and then immediately release the BindGroup object again (the WebGPU C-API
uses COM-style manual reference counting for object lifetime control, it's
valid to release the BindGroup object immediately because WebGPU
also keeps a reference for as long as the object is 'in flight').

Early versions of the sokol-gfx WebGPU backend actually had this dumb
implementation, but recently I implemented a simple bindgroups-cache which
reuses existing BindGroup objects instead of creating and discarding them in
each sg_apply_bindings() call (which would also incur significant pressure on
the Javascript garbage collector, in the first 'dumb' version I actually saw
the typical GC pauses in microbenchmark code which did thousands of resource
binding updates per frame).

The implementation details of the bind-groups-cache may differ in future
updates, but the current version is simple and straightforward instead of
trying to be clever. The cache is essentially a simple hash-indexed array using
the lower bits of a 64-bit murmur-hash computed from an array of sokol-gfx
object handles as index. A cache miss occurs if an indexed slot isn't occupied
yet, a cache collision occurs if an indexed slot already contains a BindGroups
object with different bindings. When such a slot-collision occurs, the old
BindGroup object is released before a new BindGroups object is written to that
cache slot.

If frequent hash collisions occur it might make sense to increase the size of
the bindgroups cache to the next power-of-2 size (this doesn't happen
automatically but must be tweaked at application start in the sg_setup() call).
I think that even such a dumb hash-array implementation is still
better than creating and releasing a BindGroup object in each
sg_apply_bindings() call (I still have to do the hard benchmarks to confirm
this though). The bindgroups cache may also become less useful in the future if
BindGroup creation in WebGPU becomes more optimized.

A new function `sg_query_frame_stats()` allows to peek into
sokol-gfx backend internals (like the number of bindgroup-cache hits, misses
and collisions).

It's also possible to disable the BindGroups cache alltogether, but this should
only be used for debugging purposes.

Stale BindGroup objects currently linger in the cache forever, even if their
associated sokol-gfx textures, buffers or samplers are destroyed. Since WebGPU
has managed object lifetimes this might be potentially expensive in terms of
memory consumption, because those stale BindGroup objects prevent their
referenced buffer, texture and sampler Javascript objects from being garbage-collected.

However, WebGPU has explicit destroy functions on buffer and texture objects
which cause the associated GPU resources to be freed, which keeps only a
(hopefully) tiny JS zombie object pinned. If this turns out to be a problem,
the best place to clean up stale objects in the bindgroups cache would be in
the sokol-gfx buffer-, texture- and sampler-destruction functions (basically go
over all cached bindgroup objects, and kill all bindgroups which reference the
currently destroyed buffer, texture or sampler).

I actually keep rolling around the idea in my head to add an equivalent to bindgroup
objects to the sokol-gfx API, but mainly because the `sg_bindings` struct is
growing quite big for a high-frequency function. It's unclear yet if this
would help to create WebGPU BindGroup objects upfront, because I wouldn't
want to tie such sokol-gfx bindgroup objects to specific shaders and
pipeline objects.

Phew... I really wished all that complexity to work around BindGroup shortcomings
wouldn't be needed in the first place.


### The "unfilterable-float-texture / nonfiltering-sampler" conundrum

WebGPU is a bit of a schizophrenic API because it is both quite convenient
but also extremely picky.

It is convenient in the way that it has few concepts to learn and
those concepts all connect well to each other forming a well-rounded
3D API without many surprises.

It is extremely picky in that it enforces input data correctness to a level
that's not been seen yet in native APIs. Native APIs often leave some areas
underspecified or as undefined behaviour for various reasons (where the
most likely reason is probably "oops we totally forgot about this specific edge case").

As a security-oriented web API, WebGPU can't afford the luxury
of under-specification or undefined-behaviour, but at the same
time it wants to be a high-performance API which moves as much
expensive validation checks out of the render loop and into the
initialization phase. As a result, WebGPU introduces some seemingly
'artifical' restrictions that don't exist in native 3D APIs.

The most prominent example is the strict upfront-validation around
texture/sampler combinations in WebGPU. It has always been the case that
certain types of textures don't work together with certain types of samplers on
certain types of GPUs, but in traditional APIs such details were often skipped
over or hidden in dark places of the documentation, it was some uncharted 'here be
dragons' territory manifesting as weird rendering artifacts or black screens.

Interestingly, this is an area where traditional OpenGL (the ancient version
where texture- and sampler-state was merged into the same object) was easier
to validate than modern APIs where textures and samplers are separate objects. If
texture- and sampler-state is wrapped in the same object, it's trivial to check
if both states are compatible with each other at texture creation time.

But in more recent 3D APIs, textures and samplers are completely separate
objects, their relationship doesn't become clear until they are
used together in texture sampling calls deep down in shader code.
And from the 3D API's point-of-view this is as 'here be dragons' territory
as it gets.

To correctly validate texture/sampler combinations, a modern (post-GL)
3D API needs to analyze shader code, look for texture sampling operations
in the shader and extract the texture/sampler pairs used in those
sampling operations (and that's what WebGPU needs to do
under the hood too, but most native 3D APIs most likely don't bother with
but just declare any invalid combination at runtime as UB).

With this texture/sampler usage information extracted from shaders,
WebGPU would now be able to check that textures and samplers
are of the expected types when applying resource bindings. But
now the other goal of WebGPU comes into play, which is to move
expensive validations out of the render loop into the initialization
phase, and that's how I guess the whole idea of predefined BindGroup
and BindGroupLayout objects came about.

(btw, it's design decisions like this why I think that WebGPU
won't be the "3D-API to end all 3D-APIs", other APIs might
have different opinions on what's the sweet spot in the
convenience/performance/correctness triangle)

But back to the original conundrum:

WebGPU introduces the concepts of 'texture sample types' and 'sampler binding types'.

Texture sample types are:

- float
- unfilterable-float
- s(igned)int
- u(nsigned)int
- depth

Sampler binding types are:

- filtering
- non-filtering
- comparison

Only some combinations of those are valid (not sure if I got that 100% right, the WebGPU spec is a bit opaque there):

- texture: float => sampler: filtering, non-filtering
- texture: unfilterable-float => sampler: non-filtering (there's a WebGPU extension to relax this though)
- texture: sint, uint => sampler: non-filtering
- texture: depth => sampler: comparison, non-filtering

There's a specific problem with unfilterable-float/nonfiltering texture/sampler combos though:

Shading languages usually have different sampler types which allows to infer some information
from the shader code, for instance in GLSL there are sampler types like:

- sampler2D => compatible with float texture-sample-types
- isampler2D => compatible with sint texture-sample-types
- usampler2D => compatible with uint texture-sample-types
- sampler2DShadow => compatible with depth texture-sample-types

Furthermore there are different sampler types for 1D, 2D, 3D, Cube, Array and multisampled textures.

Alltogether this provides plenty of reflection information that can be
extracted from the shader code to figure out the required texture and sampler
binding restrictions for validation on the CPU side.

Notably missing is a sampler type specifically for unfilterable-float textures (and from what
I'm seeing, WGSL doesn't have something similar either).

On the shader level this basically means that the same shader would work both
with float and unfilterable-float textures, **as long as** the texture is used
with a compatible sampler type. But in this case the reflection information
from the shader doesn't help with setting up the WebGPU BindGroupLayout objects
which require an upfront decision what texture and sampler types will be
provided.

The sokol-shdc shader compiler extracts reflection data from the shader: number and
size of uniform blocks, number and type of textures and samplers, and actually
used texture/sampler pairs. This reflection information is then passed into
the `sg_make_shader()` calls via a code-generated `sg_shader_desc` struct. The
shader code cannot provide any information about whether a provided texture
will be of the unfilterable-float type though, but WebGPU needs this information
to create BindGroupLayout objects.

I worked around this problem in sokol-shdc by adding two meta-tags which
provide a type-hint for textures and samplers, the interface reflection code then
knows that a specific texture or sampler expects the unfilterable-float /
nonfiltering flavour for a float sampling operation:

```glsl
@image_sample_type joint_tex unfilterable_float
uniform texture2D joint_tex;
@sampler_type smp nonfiltering
uniform sampler smp;
```

This hints to sokol-gfx that an 'unfilterable-float'-type texture must be bound
to 'joint_tex', and 'non-filtering'-type sampler must be bound to 'smp'.

It's a bit awkward because those meta-tags are not directly integrated with
GLSL (for that I would need to write my own GLSL parser, which is clearly overkill),
but since this hint is rarely needed I think the solution is acceptable for now.


### The Viewport Clipping Confusion

I stumbled over this pretty late in my WebGPU backend work because it's quite
subtle: much to my surprise, WebGPU currently requires viewport rectangles to be
entirely contained within the framebuffer.

None of the native backend APIs require this, so it's curious how
such a restriction slipped into WebGPU. I think this must have been
a confusion because Metal requires scissor rects (but not viewport rects)
to be contained within the framebuffer, and apparently early versions of the Metal
API documentation also were confused about viewport vs scissor
rectangles here and there (see: [https://github.com/gpuweb/gpuweb/issues/373](https://github.com/gpuweb/gpuweb/issues/373)).

Sokol-gfx allows scissor rectangles to reach outside the framebuffer, and in
the Metal and WebGPU backends the scissor rectangle is clipped to the framebuffer dimensions.
Since the scissor discard happens on the pixel level, behaviour
is identical with backend APIs like GL or D3D11 which don't have this restriction.

For viewports the situation isn't so simple though. A viewport rectangle always
maps to clipspace x/y range [-1, +1], which means changing the shape of the
viewport rectangle (for instance when clipping the rectangle against the
framebuffer boundaries) will warp the vertex-to-screenspace transform and cause
rendering to become distorted (unless the vertex transform in the vertex shader
counters the distortion, for instance with a modified projection matrix).

Of course in a wrapper API like sokol-gfx such vertex shader patching stuff
is out of the question, so what I'm currently doing is to clip the
viewport rectangle against the framebuffer, fully aware that this
will cause distortions that are not present with the other sokol-gfx backends.

This problem really needs to be fixed in WebGPU.

### Missing Features

The following sokol-gfx features are currently not supported by the WebGPU backend:

- `SG_VERTEXFORMAT_UINT10_N2`: WebGPU currently doesn't have a matching vertex format,
  but this is currently being implemented (see: [https://github.com/gpuweb/gpuweb/issues/4275](https://github.com/gpuweb/gpuweb/issues/4275))
- The following sokol-gfx pixel formats have no equivalent in WebGPU (I guess I should
  remove PVRTC support anyway, not sure yet what to do about those missing 16-bit formats though):
  - `SG_PIXELFORMAT_R16`
  - `SG_PIXELFORMAT_R16SN`
  - `SG_PIXELFORMAT_RG16`
  - `SG_PIXELFORMAT_RG16SN`
  - `SG_PIXELFORMAT_RGBA16`
  - `SG_PIXELFORMAT_RGBA16SN`
  - `SG_PIXELFORMAT_PVRTC_RGB_2BPP`
  - `SG_PIXELFORMAT_PVRTC_RGB_4BPP`
  - `SG_PIXELFORMAT_PVRTC_RGBA_2BPP`
  - `SG_PIXELFORMAT_PVRTC_RGBA_4BPP`
- And vice-versa, sokol-gfx doesn't currently support the WebGPU ASTC compressed pixel formats
  (only BCx and ETC2)
- Not directly related to sokol-gfx or the WebGPU spec: The Emscripten WebGPU
  shim currently has a couple of issues (some more, some less critical) which
  I'll keep an eye on, or may also be able to help fixing: [https://github.com/emscripten-core/emscripten/issues?q=is%3Aopen+is%3Aissue+label%3Awebgpu](https://github.com/emscripten-core/emscripten/issues?q=is%3Aopen+is%3Aissue+label%3Awebgpu)

## How sokol-gfx functions map to WebGPU functions

Here's what's happening under the hood in the sokol-gfx WebGPU backend:

- **sg_setup()**:
  - Creates one `WGPUBuffer` with usage `Uniform|CopyDst` big enough to receive all uniform
    updates for one frame.
  - Creates one `WGPUBindGroup` object to bind the uniform buffer to the vertex- and
    fragment-shader stage. The uniform buffer BindGroup will be bound to the `@group` slot 0,
    with the first four `@binding` slots assigned to the vertex stage, and the following
    four `@binding` slots assigned to the fragment stage, allowing up to four different uniform update
    'frequencies' per stage.
  - an 'empty' `WGPUBindGroup` object is created, for render pipelines that don't expect any texture and
    sampler bindings (I actually need to check if I can drop this empty bind group object)
  - an initial `WGPUCommandEncoder` object will be created for the first frame (alternatively
    this could also happen in the first render pass of a frame)

- **sg_make_buffer()**:
  - Creates one `WGPUBuffer`, the buffer size is rounded up to a multiple of 4
    (buffer size must be a multiple of 4 in WebGPU).
  - If this is an immutable buffer, the WebGPU buffer will be created as
    'initially mapped' and the initial content will be copied into the buffer
    via `wgpuBufferGetMappedRange()` + `memcpy()` + `wgpuBufferUnmap()` (this is
    current fairly inefficient on WASM because it involves a redundant heap
    allocation and data copy in the Javascript shim, this will probably be
    optimized in a followup update)

- **sg_destroy_buffer()**:
  - Calls `wgpuBufferDestroy` and `wgpuBufferRelease`. The first call explicitly destroys any
    GPU resources associated with the buffer, leaving behind a small Javascript 'zombie object'
    which is subject to JS garbage collection. The `wgpuBufferRelease` call decrements a reference
    count which is used for lifetime management on native platforms, and in the JS shim to
    pin the associated WebGPU object on the Javascript side.

- **sg_make_image()**:
  - Creates one `WGPUTexture` object and one `WGPUTextureView` object for accessing
    the texture from a shader stage.
  - If this is an immutable texture, the initial data will be copied into the
    texture with a series of `wgpuQueueWriteTexture()` calls

- **sg_destroy_image()**:
  - Calls `wgpuTextureViewRelease`, `wgpuTextureDestroy` and `wgpuTextureRelease`

- **sg_make_sampler()**:
  - creates one `WGPUSampler` object

- **sg_destroy_sampler()**:
  - calls `wgpuSamplerRelease`

- **sg_make_shader()**:
  - creates two `WGPUShaderModule` objects (one per shader stage) from the
    WGSL vertex- and fragment-shader source code provided in the
    `sg_shader_desc` struct
  - creates one `WGPUBindGroupLayout` from the shader interface reflection info provided
    in `sg_shader_desc`, texture/sampler bindings go into `@group` slot 1 and at predefined
    `@binding` slots:
    - vertex shader textures start at `@group(1) @binding(0)`
    - vertex shader samplers start at `@group(1) @binding(16)`
    - fragment shader textures start at `@group(1) @binding(32)`
    - fragment shader samplers start at `@group(1) @binding(48)`

- **sg_destroy_shader()**:
  - calls `wgpuBindGroupLayoutRelease` and 2x `wgpuShaderModuleRelease`

- **sg_make_pipeline()**:
  - creates one `WGPUPipelineLayout` object which merges the global uniform buffer
    binding with the pipeline shader's texture/sampler bindings
  - creates one `WGPURenderPipeline` object
  - the separate `WGPUPipelineLayout` object is no longer needed and immediate released

- **sg_destroy_pipeline()**:
  - calls `wgpuRenderPipelineRelease`

- **sg_make_pass()**
  - creates one `WGPUTextureView` object for each pass attachment (color-, resolve- and
    depth-stencil-attachments), this may result in up to 9 texture-view objects

- **sg_release_pass()**:
  - one call to `wgpuTextureViewRelease` for each texture-view object created in
    sg_make_pass() (up to 9x)

- **sg_begin_pass()**
  - calls `wgpuCommandEncoderBeginRenderPass` which returns a reference to a `WGPURenderPassEncoder` object
  - calls `wgpuRenderPassEncoderSetBindGroup` to apply an empty bind group for textures and samplers
    (need to check if there's a better solution)
  - calls `wgpuRenderPassEncoderSetBindGroup` to bind the global uniform buffer (something must
    be bound to the uniform buffer bind slots, even if the pass doesn't require uniform data)

- **sg_apply_viewport()**
  - clips the viewport to the current framebuffer boundaries (which is actually wrong, but currently required
    by WebGPU), and then calls `wgpuRenderPassEncoderSetViewport`

- **sg_apply_scissor_rect()**
  - clips the scissor rect to the current framebuffer boundaries and then calls `wgpuRenderPassEncoderSetScissorRect`

- **sg_apply_pipeline()**
  - calls `wgpuRenderPassEncoderSetPipeline`
  - calls `wgpuRenderPassEncoderSetBlendConstant`
  - calls `wgpuRenderPassEncoderSetStencilReference`

- **sg_apply_bindings()**
  - may call `wgpuRenderPassEncoderSetIndexBuffer` (if provided and the same index buffer isn't already bound)
  - up to 8x may call `wgpuRenderPassEncoderSetVertexBuffer` (if provided and the same vertex buffer isn't already bound at that slot)
  - may call `wgpuDeviceCreateBindGroup` on a bindgroups-cache miss
  - may call `wgpuBindGroupRelease` on a bindgroups-cache slot collision
  - may call `wgpuRenderPassEncoderSetBindGroup` if the same bindgroup isn't already bound

- **sg_apply_uniforms()**
  - calls `wgpuRenderPassEncoderSetBindGroup` on the global uniform buffer to update one dynamic offset (out of 8)

- **sg_wgpu_draw()**
    - either calls `wgpuRenderPassEncoderDrawIndexed` or `wgpuRenderPassEncoderDraw`

- **sg_end_pass()**:
  - calls `wgpuRenderPassEncoderEnd` and `wgpuRenderPassEncoderRelease`

- **sg_commit()**
  - calls `wgpuQueueWriteBuffer` to copy the frame's uniform data from the WASM heap
    into the WebGPU uniform buffer
  - calls `wgpuCommandEncoderFinish` which returns a reference to a `WGPUCommandBuffer` object
  - calls `wgpuCommandEncoderRelease`
  - calls `wgpuQueueSubmit` with the command-buffer reference, followed by `wgpuCommandBufferRelease`
  - finally call `wgpuDeviceCreateCommandEncoder` for the next frame (probably makes more sense to
    move this into the first begin-pass of a frame though)

- **sg_update_buffer() / sg_append_buffer()**
    - calls `wgpuQueueWriteBuffer` once or twice (twice if the data size isn't
      a multiple of 4, in that case the dangling 1..3 bytes need to be copied
      separately as a single 4-bytes block)

- **sg_update_image()**
    - copies the data as a series of `wgpuQueueWriteTexture()` calls (same code
      that's used for populating immutable textures in `sg_make_image()`)


## What's next for sokol-gfx

Apart from the usual bugfixing and maintenance the following things are on
the long-term sokol-gfx roadmap (in undefined order):

- a 'begin-pass' unification which makes rendering into different externally
  provided swapchains easier (see: [https://github.com/floooh/sokol/issues/904](https://github.com/floooh/sokol/issues/904))
- a cleanup and 'orthoganalization' of the currently very restricted resource
  update functions: cpu-to-gpu and gpu-to-gpu copy functions, and ideally a non-stalling
  gpu-to-cpu), all this needs performance-investigations in the GL backend though
- initial storage-buffer-support which allows a cleaner way to access structured
  data in shaders (as opposed to the current workaround of uploading such
  data in textures) - this will need to leave WebGL2 behind though
- initial compute-shader support (same: no WebGL2 support)
- maybe (just maybe) switch from GLSL to WGSL as the primary cross-backend
  shader authoring language, this assumes that Tint can entirely replace
  SPIRVCross, which is not certain yet

...for the rest of the year I will probably tinker with other things though (I have the
urge to do some emulator coding again, and I also have an unfinished experiment
from last spring lying around for a fips cmake-wrapper replacement)

Over and out :)
