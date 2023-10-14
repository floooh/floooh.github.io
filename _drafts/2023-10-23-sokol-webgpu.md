---
layout: post
title: "The new sokol-gfx WebGPU backend"
---

In a couple of days I will merge the new WebGPU backend in sokol_gfx.h, and instead of an oversized
changelog entry I guess it's time for a new blog post instead (the changelog entry will have all the
technical details too, but here I want to go a bit beyond that and also talk about the design decisions,
what went well and didn't and what to expect in the future).

The sokol WebGPU backend samples [are hosted here](https://floooh.github.io/sokol-webgpu/).

However, the source-code links will point to outdated code until the WebGPU branch is merged to master.

## WebGPU in a Nutshell

From the perspective of sokol-gfx, WebGPU is a fairly straightforward API:

- a WGPUDevice object which creates other WebGPU objects
- WGPUQueue, WGPUCommandBuffer and WGPUCommandEncoder as general
  infrastructure for sending commands to the GPU
- WGPURenderPassEncoder to record commands for one render pass
  into a command encoder
- WGPUBuffer objects to hold vertex-, index- and uniform-data
- WGPTexture objects to hold pixel data in 'subimages', and WGPUTextureView
  objects for defining a group of texture subimages.
- WGPUSampler objects for describing how pixel data is sampled in shaders
- WGPUShaderModule objects which encapsulate shader code
- WGPUBindGroupLayout and WGPUPipelineLayout which together describe
  an interface for how shader-resource-objects (uniform-buffers, texture-views
  and samplers) are communicated to shaders
- WGPUBindGroup for storing immutable shader-resource-object combinations
- WGPURenderPipelineState to combine a vertex layout, shaders, a resource
  binding interface and granular render states into a single immutable
  state object

...and that's about it. Currently sokol-gfx doesn't use the following WebGPU
features:

- storage buffers and storage textures
- compute passes
- render bundles
- occlusion and timestamp queries

WebGPU will slowly replace WebGL2 as the 'reference platform' to define the
future sokol feature set. However, only for features that can also be
implemented without an emulation layer on top of D3D11, Metal and desktop GL.

Storage resource and compute passes are definitely at the top of the list,
while the idea of render bundles will most likely never make it into sokol-gfx.

## WebGPU in the Sokol Ecosystem

As with other backends, **sokol_gfx.h** expects that the `WGPUDevice` and
swapchain resources are created and managed externally. sokol-gfx
only depends on [<webgpu/webgpu.h>](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h).

Initially, **sokol_app.h** will only support setting up a WebGPU device
and swapchain for the Emscripten backend as alternative to WebGL2.
This means that another 'window system glue' library (like GLFW) must be used
when sokol-gfx is used together with a native WebGPU library like
[Google's Dawn](https://dawn.googlesource.com/dawn).

The **sokol-shdc** shader compiler gains support for translating Vulkan-style
GLSL (where 'Vulkan-style' means 'separate texture and sampler objects) via
SPIRV into WGSL with the help of [Google's
Tint](https://dawn.googlesource.com/tint) library.

## The Gnarly Parts

Most of the sokol-gfx WebGPU backend is a straightforward data- and function-mapping,
there are some notable exceptions though:

### Uniform Data Updates

sokol-gfx doesn't expose uniform buffers to the API user, instead uniform data
is considered to be 'frame-transient', all uniform data required for one frame
must be written from scratch via sg_apply_uniforms() calls (this make sense
because most uniform data rarely remains unchanged between frames). Each
sokol-gfx shader stage offers 4 uniform data 'slots' which allows to supply
uniform data at different update frequencies.

At startup, the sokol-gfx WebGPU backend creates a single uniform buffer which
must be big enough to hold all uniform data for one worst-case frame.
Additionally, an intermediate memory buffer of the same size is allocated on
the heap.

When new uniform data is coming in via `sg_apply_uniforms()`, the data is
appended to the intermediate memory buffer and an offset to the new data is
recorded into the render pass encoder by calling `wgpuRenderPassSetBindGroup`
with dynamic buffer offsets (where only one of 8 offsets has actually changed).

At the end of the frame in `sg_commit()` a single big `wgpuQueueWriteBuffer` records
a write operation from the intermediate WASM heap buffer into the WebGPU uniform
buffer.

This uniform update mechanism works similar to the native Metal backend, but
with two important differences: In the Metal backend, there is no intermediate
CPU memory buffer, the data is written directly into one of multiple uniform
buffers which are rotated through. For updating the uniform buffer offset,
Metal offers special 'slim' functions to record only a single buffer offset
update for an already bound buffer.

I went through several rewrites of the uniform update code, and the current
version most likely isn't the last one (it's good enough for the initial
implementation though):

- The first version implemented a 'buffer conveyor belt' model, this basically
  rotates through multiple buffers, where the current write buffer is mapped
  for the duration of a frame. This version is quite similar to the Metal
  backend. The problem when using WebGPU from WASM however is that a WebGPU
  buffer cannot be mapped directly into the WASM heap. The result of mapping
  a WebGPU buffer is a Javascript ArrayBuffer object, while the WASM heap
  is a separate ArrayBuffer objects. A C-API shim on top of WASM needs to
  go through a temporary heap allocation in the WASM heap to emulate the
  WebGPU buffer mapping functions, which of course is inefficient.

- The next uniform-update rewrite called a new `wgpuQueueWriteBuffer` function
  once per sg_apply_uniform() call. This writeBuffer function conveniently
  implemenets the buffer-conveyor-belt under the hood, and the JS shim function
  also doesn't need a temporary WASM heap allocation. It turned out that the
  writeBuffer call overhead was too high for such a high-frequency operation...

- And thus the current version was created, which accumulates all uniform
  data snippets for an entire frame and then does a single big writeBuffer()
  call at the end of the frame in sg_commit().

- There's another option though which I will try out at a later time (because
  that intermediate memory buffer bother me), but this method requires different
  code paths for WASM vs native platforms: This method would go back to
  a manually implemented buffer-conveyor-belt, but call out to a handful
  of my own Javascript shim functions. At the start of a frame, a JS
  function would be called which obtains a mapped ArrayBuffer from the
  current frame's uniform buffer and that ArrayBuffer would be stored
  somewhere until the end of the frame. During the frame the WebGPU
  backend version of sg_apply_uniforms() would call out into
  another Javascript function which directly copies the uniform
  data snippet from the WASM heap ArrayBuffer object into the
  mapped uniform ArrayBuffer. Finally at the end of the frame,
  a third Javascript shim function is called which unmaps the uniform
  ArrayBuffer. On native platforms, the regular WebGPU buffer mapping
  functions would be used instead.

The next problem with WebGPU uniform buffer updates is unfixable on my side though:

Apart from copying the uniform data snippets into a uniform buffer, sg_apply_uniforms()
also needs to record the offset of that data in the uniform for the next draw call.

In WebGPU this requires a setBindGroup() call with so-called 'dynamic offsets'.

And currently, this setBindGroup() call has a surprisingly high CPU overhead which
makes the `sg_apply_uniforms()` call in the WebGPU significantly slower than in
the WebGL2 backend. How big the difference is depends on the platform, but as
an example: on my Windows PC in Chrome with WebGL2 I can issue about 64k
uniform-update/draw-call pairs before frame rate starts to drop below 60Hz,
while on WebGPU it tops out at around 16k uniform-update/draw-call pairs.

The culprit is the setBindGroup() call which is a lot slower than the
glUniform4fv() call that's used in the WebGL2 backend to update uniform data.

You can check for yourself in the following demo for
[WebGL2](https://floooh.github.io/sokol-html5/drawcallperf-sapp.html) and
[WebGPU](https://floooh.github.io/sokol-webgpu/drawcallperf-sapp.html).

To me it was definitely surprising that there are situations where WebGPU
can be drastically slower than WebGL2, and hopefully this is just a case
of "first make it work, then make it fast". But it looks like moving
from WebGL2 to WebGPU isn't such a no-brainer as it was on the iOS and
macOS platforms (where simply switching from GL to Metal provides a
hefty performance increase).


### Texture and Sampler Resource Bindings

Another BindGroup-related problem, what a surprise ;)

This time it's about the texture and sampler resource bindings.

Just as with uniform data, sokol-gfx considers shader resource bindings to be
frame-transient, e.g. they need to be defined from scratch each frame. The
motivation for this isn't quite as clear-cut as for uniform data though. In
games for instance, material systems can often stamp out all required material
instances upfront. But especially in non-game-application resource binding
combinations are often unpredictable, which can lead to combinatorial
explosions when trying to create all required combinations upfront.

Resource binding in sokol-gfx happens through the `sg_apply_bindings()` call
which takes separate arrays of textures and samplers for each shades stage.

In WebGPU, those textures and samplers must be baked into a BindGroup object.
This means that a dumb sokol-gfx backend would simply create a BindGroup object
inside `sg_apply_bindings()`, call setBindGroup() on the render pass encoder,
and then immediate release the BindGroup object again (the WebGPU C-API
uses COM-style manual reference counting for object lifetime control).

Early versions of the WebGPU backend actually had this dumb implementation,
but for the initial release I implemented a simple bind-groups-cache which
try to reuse existing BindGroup objects instead of creating and discarding
(which would also incur significant pressure on the Javascript garbage collector,
in the first 'dumb' version I actually saw the typical GC pauses in microbenchmark
code which did thousands of resource binding updates per frame).

The implementation details of the bind-groups-cache may differ, but the current
version is simple and straightforward instead of trying to be clever. The cache
is essentially a simple hash-indexed array using the lower bits of a 64-bit
murmur-hash of sokol-gfx object handles as index. A cache miss occurs if either
an indexed slot isn't occupied yet, or that slot already contains a BindGroups
object with different bindings. When such a slot-collision occurs, the old
BindGroups object is released and a new BindGroups object is written to that
cache slot.

If frequent hash collisions occur it might make sense to increase the size
of the bindgroups cache (this doesn't happen automatically but must be
tweaked at application start in the sg_setup() call).

A new function `sg_query_frame_stats()` allows to peek into
sokol-gfx backend internals (like the number of bindgroup-cache hits, misses
and collisions).

I actually keep rolling around the idea in my head to add an equivalent to bindgroup
objects to the sokol-gfx API, but mainly because the `sg_bindings` struct is
growing quite big for a high-frequency function. It's unclear yet if this
would help to create WebGPU BindGroup objects upfront, because I wouldn't
want to tie such sokol-gfx bindgroup objects to specific shaders and
pipeline objects.


### The "unfilterable-float-texture / nonfiltering-sampler" conundrum

WebGPU is a bit of a schizophrenic API because it is both convenient
but also extremely picky.

It is convenient in the way that it has few concepts to learn and
those concepts all connect well to each other forming a well-rounded
3D API without many surprises.

It is extremely picky in that it enforces input data correctness to a level
that's not been seen yet in native APIs. Native APIs often leave some areas
underspecified or as undefined behaviour for various reasons (one reason
is most likely plain and simple "oops we totally forgot about this edge case").

As a security-oriented web API, WebGPU can't effort the luxury
of under-specification or undefined behaviour, but at the same
time it wants to be a high-performance API which moves as much
expensive validation checks out of the render loop and into the
initialization phase. As a result, WebGPU introduces some seemingly
'artifical' restrictions that don't exist in native 3D APIs.

The most prominent example is the strict validation around
texture/sampler combinations in WebGPU. It has always been the
case that certain types of textures don't work together with
certain types of samplers on certain types of GPUs, but in traditional
APIs such details were often skipped over in 3D API documentations,
it was some uncharted 'here be dragons' territory manifesting as
weird rendering artifacts or black screens.

Interestingly, this is an area where traditional OpenGL (the ancient
version where texture objects also contained the sampler state) were
more correct than modern APIs where textures and samplers are separate
objects. If texture- and sampler-state is wrapped in the same object,
it's trivial to check wether both are compatible.

But in modern 3D APIs, textures and samplers are completely separate
objects, their relationship doesn't become clear until they are
used together in texture sampling calls deep down in a shader functions.
And from the 3D APIs point-of-view this is as 'here be dragons' territory
as it gets.

To correctly validate texture/sampler combinations, a modern (post-GL)
3D API needs to analyze shader code, look for texture sampling functions
in the shader code and extract the texture and sampler used in each
those functions (and that's what WebGPU needs to do
under the hood, but native APIs most likely don't bother with).

With this texture/sampler usage information extracted from shaders,
WebGPU would now be able to check that textures and samplers
are of the expected types when applying resource bindings. But
now the other goal of WebGPU comes into play, which is to move
expensive validations out of the render loop into the initialization
phase, and that's how I guess the whole idea of BindGroup
and BindGroupLayout objects came about.

(btw, it's design decisions like this why I think that WebGPU
won't be the "3D-API to end all 3D-APIs", other APIs might
have different opinions on what's the sweet spot in the
convenience vs performance vs correctness triangle)

But were far from finished here.

WebGPU introduces the concepts of 'texture sample types' and
'sampler binding types'.

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

Only some combinations of those are valid (not sure if I got that 100% right though, the WebGPU spec is a bit opaque there):

- float => filtering, non-filtering
- unfilterable-float => non-filtering (there's a WebGPU extension to relax this though)
- sint, uint => non-filtering
- depth => comparison, non-filtering

There's a specific problem with unfilterable-float/nonfiltering texture/sampler combos though:

Shading languages usually have different sampler types which allows to infer some information
from the shader code, for instance in GLSL there are sampler types like:

- isampler2D => compatible with sint texture-sample-types
- usampler2D => compatible with uint texture-sample-types
- sampler2D => compatible with float texture-sample-types
- sampler2DShadow => compatible with depth texture-sample-types

Furthermore there are different sampler types for 1D, 2D, 3D, Cube, Array and multisampled textures.

This provides plenty of reflection information to figure out the required texture and sampler types
for validation on the CPU side.

Notably missing is a sampler type specifically for unfilterable-float textures (and from what
I'm seeing, WGSL doesn't have something similar either).

On the shader level this basically means that the same shader works both with
float and unfilterable-float textures, **as long as** it is used with a compatible
sampler type. But in this case the reflection information from the shader
doesn't help with setting up the BindGroupLayouts (because that's exactly
that sokol-gfx in combination with the sokol-shdc offline shader-compiler does).

For the sokol-shdc shader compiler, I worked around this problem by adding to
meta-tags which provide this hint for textures and samplers, the interface
reflection code then knows that a specific texture or sampler expects the
unfilterable-float / nonfiltering flavour for a floating point sampling
operation:

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

None of the native backend APIs have this restrictions, so it's curious how
such a restriction slipped into WebGPU. I think this must have been
a confusion because Metal requires scissor rects (but not viewport rects)
to be contained within the framebuffer, and early versions of the Metal
API documentation also seems to have confused viewport and scissor
rectangles.

sokol-gfx allows scissor rectangles to reach outside the framebuffer, and in
the Metal backend the scissor rectangle is clipped to the framebuffer dimensions.
Since the scissor discard happens on the fragment/pixel level, behaviour
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