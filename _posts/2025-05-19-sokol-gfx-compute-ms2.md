---
layout: post
title: The sokol-gfx 'compute milestone 2' update
---

In a couple of days I will merge the next breaking sokol_gfx.h update (aka the `compute-ms2`
update) which makes working with buffer objects a bit more flexible and will allow
compute shaders to write to `sg_image` objects via 'compute pass attachments'.

The update also comes with a matching sokol-shdc update which writes additional
reflection information for storage images used in compute shaders into the
code-generated `sg_shader_desc` struct.

> NOTE: all WASM sample URLs in the blog post require a WebGPU capable browser and will only
> be valid after the merge.

The implementation ticket is here, and this also has links to all related
PRs: [https://github.com/floooh/sokol/issues/1244](https://github.com/floooh/sokol/issues/1244)

### Updated documentation sections

- in sokol_gfx.h, re-read the updated section [ON COMPUTE PASSES](https://github.com/floooh/sokol/blob/afc74bd88eab597665f5e4f10962c73524d7cbc1/sokol_gfx.h#L707-L798)
- if you're not using sokol-shdc for shader compilation, also re-read the
  updated section [ON SHADER CREATION](https://github.com/floooh/sokol/blob/afc74bd88eab597665f5e4f10962c73524d7cbc1/sokol_gfx.h#L801-L1036)
  (most of that information is only needed when *not* using sokol-shdc though)
- read the new doc section [ON STORAGE IMAGES](https://github.com/floooh/sokol/blob/afc74bd88eab597665f5e4f10962c73524d7cbc1/sokol_gfx.h#L1390-L1436)

### An important behaviour change for immutable buffer objects

The initial 'compute shader' update allowed to create immutable buffers without
initial data and guaranteed that the buffer content would be zero-initialized.
On some backend APIs this required a temporary memory allocation of the buffer
size which obviously wasn't great.

This guaranteed zero-initialization has been rolled back now and the rules
for creating immutable buffer objects have been changed like this:

- when creating an immutable non-storage-buffer object (e.g. the buffer cannot
  be written to with a compute shader), initial data *must* be provided
- when creating an immutable storage-buffer object, no initial data needs to
  provided, but in that case the buffer content will be 'undefined'

In practice this means that when you use a compute shader to initialize
storage buffer content you can no longer rely on the initial buffer content being
zero-initialized, instead write *all* buffer items in the compute shader,
even when they are supposed to be zero.

### Multi-purpose buffer objects

It's now possible to bind the same buffer object to different bind points (e.g.
bind the same buffer as vertex buffer, index buffer and/or storage buffer).
This means the following scenarios are now enabled:

- It's possible to stash vertices and indices into the same buffer
  (with the exception of WebGL2 where this is explicitly disallowed)
- It's now possible to use a compute shader to write data to a buffer, and
  then bind this buffer as vertex- or index-buffer.

To achieve this, the `sg_buffer_desc` struct has been changed to merge the previous
buffer type and buffer usage enum items into a new `sg_buffer_usage` struct which is a
boolean flag group:

```c
typedef struct sg_buffer_usage {
    bool vertex_buffer;
    bool index_buffer;
    bool storage_buffer;
    bool immutable;
    bool dynamic_update;
    bool stream_update;
} sg_buffer_usage;
```

The default setup configures an immutable vertex buffer (just as before), e.g.
creating a buffer object like this:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .data = SG_RANGE(vertices),
})
```

...is identical with:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = {
        .vertex_buffer = true,
        .immutable = true,
    },
    .data = SG_RANGE(vertices),
});
```

...to create an immutable index buffer:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = {
        .index_buffer = true,
    },
    .data = SG_RANGE(indices),
});
```

...to create an index buffer with stream-update hint:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = {
        .index_buffer = true,
        .stream_update = true,
    },
});
```

...to create a buffer that can be written by a compute shader and then
bound to a vertex buffer bindpoint:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = {
        .vertex_buffer = true,
        .storage_buffer = true,
    },
});
```

...and the same as index buffer:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = {
        .index_buffer = true,
        .storage_buffer = true,
    },
});
```

To stash both vertices and indices into the same buffer object:

```c
const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = {
        .vertex_buffer = true,
        .index_buffer = true,
    },
    .data = SG_RANGE(vertices_and_indices),
});
```

Note that 'multi-purpose buffer usage' is explicitly disallowed on WebGL2 (which is
only relevant for using a single buffer to hold vertex- and index-data, since
storage buffers are not available on WebGL2 anyway). To check for this restriction
use the new `sg_features.separate_buffer_types` boolean:

```c
if (sg_query_features().separate_buffer_types) {
    const sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
        .usage = {
            .vertex_buffer = true,
            .index_buffer = true,
        },
        .data = SG_RANGE(vertices_and_indices),
    });
}
```

Any invalid combination of usage flags will also be checked in the sokol-gfx validation layer.

The following new sample uses a combined vertex/index buffer:

- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.glsl](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.glsl)
- WASM: [https://floooh.github.io/sokol-webgpu/vertexindexbuffer-sapp-ui.html](https://floooh.github.io/sokol-webgpu/vertexindexbuffer-sapp-ui.html)

The `instancing-compute-sapp` sample has been updated to bind the compute-shader-updated
storage buffer as vertex buffer with hardware instancing:

- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.glsl](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.glsl)
- WASM: [https://floooh.github.io/sokol-webgpu/instancing-compute-sapp-ui.html](https://floooh.github.io/sokol-webgpu/instancing-compute-sapp-ui.html)

There is no sample yet which uses a compute shader to write index data.

### Breaking changes when creating image objects

Similar to the above `sg_buffer_desc` change, usage hints in the `sg_image_desc` struct
are now provided through a new `sg_image_usage` struct looking like this:

```c
typedef struct sg_image_usage {
    bool render_attachment;
    bool storage_attachment;
    bool immutable;
    bool dynamic_update;
    bool stream_update;
} sg_image_usage;
```

E.g. creating a 'render-target texture' for offscreen rendering now looks like this:

```c
const sg_image img = sg_make_image(&(sg_image_desc){
    .usage = {
        .render_attachment = true,
    },
    ...
});
```

...and creating a image updated dynamically with CPU data with stream-update
behaviour:

```c
const sg_image img = sg_make_image(&(sg_image_desc){
    .usage = {
        .stream_update = true,
    },
    ...
});
```

As with `sg_buffer_usage`, invalid usage flag combinations are caught in the
sokol-gfx validation layer.


### Compute pass attachments (aka storage images)

It's now possible to use compute shaders to write to `sg_image` objects. The way this is currently
implemented is very similar to offscreen rendering (but will change in a future 'resource view update',
more info on that at the end of the blog post).

Let's first write a simple compute shader in the sokol-shdc GLSL flavour which writes some
animated color gradient to a storage image:

```glsl
@cs cs
layout(binding=0) uniform cs_params {
    float offset;
};
layout(binding=0, rgba8) uniform writeonly image2D cs_out_tex;
layout(local_size_x=16, local_size_y=16) in;

void main() {
    ivec2 size = imageSize(cs_out_tex);
    ivec2 pos = ivec2(mod(vec2(gl_GlobalInvocationID.xy) + vec2(size) * offset, size));
    vec4 color = vec4(vec2(gl_GlobalInvocationID.xy) / float(size), 0, 1);
    imageStore(cs_out_tex, pos, color);
}
@end
@program compute cs
```

On the CPU side, create an `sg_image` object with 'storage attachment usage':

```c
const sg_image img = sg_make_image(&(sg_image_desc){
    .usage = {
        .storage_attachment = true,
    },
    .width = WIDTH,
    .height = HEIGHT,
    .pixel_format = SG_PIXELFORMAT_RGBA8,
});
```

Next the image must be wrapped in an `sg_attachments` object. This allows to pick a specific
image surface (mip-level and/or slice) for the compute shader to access.
Up to 4 (or `SG_MAX_STORAGE_ATTACHMENTS`) images can be defined in a single attachment:

```
const sg_attachments atts = sg_make_attachments(&(sg_attachments_desc){
    .storages[SIMG_cs_out_tex] = {
        .image = img,
        // optionally pick a mip level and slice:
        .mip_level = 0,
        .slice = 0,
    },
});
```

...next a compute pipeline object which wraps the above compute shader:

```c
const sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .compute = true,
    .shader = sg_make_shader(compute_shader_desc(sg_query_backend)),
});
```

In the frame loop, run a compute pass and provide the attachments object,
apply the compute pipeline and uniform data, and finally call `sg_dispatch()`
to kick off the compute shader:

```c
sg_begin_pass(&(sg_pass){ .compute = true, .attachments = atts });
sg_apply_pipeline(pip);
sg_apply_uniforms(UB_cs_params, &SG_RANGE(cs_params));
sg_dispatch(WIDTH / 16, HEIGHT / 16, 1);
sg_end_pass();
```

...after the compute pass the image object can then be used as a texture binding in a regular render pass.

Find the complete sample here:

- WASM: [https://floooh.github.io/sokol-webgpu/write-storageimage-sapp.html](https://floooh.github.io/sokol-webgpu/write-storageimage-sapp.html)
- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.glsl](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.glsl)

...and a more advanced example which has been ported from WebGPU:

- WASM: [https://floooh.github.io/sokol-webgpu/imageblur-sapp.html](https://floooh.github.io/sokol-webgpu/imageblur-sapp.html)
- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c)

### Detailed change list

#### sokol_app.h:

The D3D11/DXGI backend now creates a `D3D_FEATURE_LEVEL_11_1` device (with a
fallback to `D3D_FEATURE_LEVEL_11_0`). Feature Level 11.1 is needed to allow
more than 8 UAV (Unordered Access View) bindings. D3D11.1 was released around
2011 with Windows 8, so this is only an issue if support for Windows 7 is still
required or on very old GPUs (Win7 is now at 0.12% on Steam Hardware Survey,
but even if this turns out to be a problem, only the bindslot allocation
strategy in sokol-shdc for HLSL5 UAV bindslots needs to be changed).

#### sokol_gfx.h:

- A new constant `SG_MAX_STORAGE_ATTACHMENTS = 4` has been added (most likely
  bumped to at least 8 in the future)
- The struct `sg_pixelformat_info` has gained two new flags:
    - `bool read`: true if the pixel format supports compute shader read access
    - `bool write`: true if the pixel format supports compute shader write access

  Currently the list of compute shader accessible pixel formats is hardwired to
  the following list which is safe to use across all GPUs and backend APIs
  (all those formats support read+write access):
    - `SG_PIXELFORMAT_RGBA8`
    - `SG_PIXELFORMAT_RGBA8SN/UI/SI`
    - `SG_PIXELFORMAT_RGBA16UI/SI/F`
    - `SG_PIXELFORMAT_R32UI/SI/F`
    - `SG_PIXELFORMAT_RG32UI/SI/F`
    - `SG_PIXELFORMAT_RGBA32UI/SI/F`

- A new feature flag `sg_features.separate_buffer_types` has been added,
  this is only true on WebGL2. The only effect of that flag is that
  the same buffer object cannot be used as vertex- and index-buffer bindings.
- The enums `sg_usage` and `sg_buffer_type` have been removed.
- The struct `sg_buffer_usage` has been added.
- The enum field `sg_buffer_desc.type` has been removed and replaced by
  boolean flags in `sg_buffer_usage`.
- The enum field `sg_buffer_desc.usage` has been repurposed as nested
  struct item of type `sg_buffer_usage`.
- The struct `sg_image_usage` has been added.
- The boolean `sg_image_desc.render_target` has been removed and replaced
  by `sg_image_usage.render_attachment`
- The enum feld `sg_image_desc.usage` has been repurposed as nested struct
  item of type `sg_image_usage`.
- A new struct `sg_shader_storage_image` has been added, this is nested in
  in `sg_shader_desc` and holds reflection information about storage image
  bindings in compute shaders.
- A new array `sg_shader_desc.storage_images[]` has been added to communicate
  reflection information about storage image usage in compute shaders to sokol_gfx.h
- A new array `sg_attachments_desc.storages[]` has been added to describe
  'storage image attachments' for compute passes.
- The function `sg_query_buffer_usage()` now returns a struct `sg_buffer_usage`.
- The function `sg_query_image_usage()` now returns a struct `sg_image_usage`.


### What's next

Long story short: while working on the storage image update it became clear
that sokol_gfx.h needs resource-view objects.

This will allow more flexible resource bindings without creating temporary
3D-backend objects in the 'hot path' while keeping the sokol_gfx.h backend
implementations simple (e.g. I want to avoid a dynamic 'hash-and-cache'
approach as much as possible, it's already bad enough that this is needed
with WebGPU BindGroups).

Currently resource view objects are managed under the hood, for instance
in the D3D11 backend:

- `sg_buffer` objects with storage buffer usage generally create an Shader Resource View
  for readonly-access in vertex-, fragment- and compute-shaders, and if the buffer
  is immutable, also an Unordered Access View for write-access in compute shaders.
  Notably, any starting offsets are hardwired to zero in both view objects.
- `sg_image` objects generally create a Shader Resource View object, but without
  allowing to specify a mip-level range, array-slice range or different pixel format.
- `sg_attachments` objects create:
    - one Render Target View object per color attachments
    - an optional Depth Stencil View object for the depth-stencil attachment
    - one Unordered Access View object per storage attachment

The reason why storage images are currently treated as pass attachments instead
of regular bindings applied via `sg_apply_bindings()` is because storage image
bindings need to pick a mip-level and/or slice, and at least on D3D11 this
requires a baked UAV object. Likewise, binding the same storage buffer with
different offsets would require one SRV or UAV object per offset.

The current plan for view objects in sokol_gfx.h looks like this:

- a single new resource object type is added: `sg_view`, with matching structs and functions
  (`sg_view_desc`, `sg_make_view()`, `sg_destroy_view()`, etc...)
- in return, the `sg_attachments` resource object type is removed (along with `sg_attachments_desc`,
  `sg_make_attachments()`, `sg_destroy_attachments()` etc...)
- view objects can be thought of as specialization of a resource object for
  a specific bindslot type (I actually thought about calling the new resource type `sg_binding`,
  but 'view' is the established name for this type of thing across backend 3D APIs), e.g. views will come in the
  following 'runtime flavours':
    - texture views
    - storage buffer views
    - storage image views
    - color attachment views
    - resolve attachment views
    - depth-stencil attachment views
- ...and maybe (but not sure yet):
    - vertex buffer views
    - index buffer views

  ...vertex- and index-buffer-views would allow to remove the bind offset for
  vertex- and index-buffers from `sg_bindings`, with the downside that one view
  object would be required per offset, but I can't think of a situation where a
  highly dynamic starting offset would be required for vertex- and index-data.
  To be clear: there is no backend API which requires a view object for vertex-
  and index-buffer bindings, it would be purely a sokol_gfx.h thing (this also
  means that it would be very cheap to build and destroy vertex- and
  index-buffer-view objects on the fly since no calls into backend APIs would happen)

- the new `sg_bindings` struct would then look like this (notably storage
  images for compute shader access would move from 'pass attachments' to
  regular 'bindings')

  ```c
  typedef struct sg_bindings {
      sg_view vertex_buffers[SG_MAX_VERTEXBUFFER_BINDINGS]
      sg_view index_buffer;
      sg_view textures[SG_MAX_TEXTURE_BINDINGS];
      sg_view storage_buffers[SG_MAX_STORAGEBUFFER_BINDINGS];
      sg_view storage_images[SG_MAX_STORAGEIMAGE_BINDINGS]
      sg_sampler samplers[SG_MAX_SAMPLER_BINDINGS];
  } sg_bindings;
  ```

- `sg_attachments` would become a 'transient struct' similar to
  `sg_bindings`:

  ```c
  typedef struct sg_attachments {
      sg_view colors[SG_MAX_COLOR_ATTACHMENTS];
      sg_view resolves[SG_MAX_COLOR_ATTACHMENTS];
      sg_view depth_stencil;
  } sg_attachments;
  ```

This 'view update' would have the following advantages:

- storage buffer bindings can have a starting offset, which simplifies
  managing different types of data in the same buffer
- texture and storage image bindings can (to some extent) reinterpret
  the image data (e.g. casting to a different pixel format or selecting
  a miplevel and slice range - this will have to be behind a feature flag
  though)
- multiple-render-target combinations no longer need to be prebaked

No ETA yet on the 'view update' though, first I want to fix a couple of
internal things:

- the GL texture creation code is currently an unholy combination of
  `glTexStorage` and `glTexImage` functions. I want to cleanly split
  this into two code paths (unfortunatly macOS being stuck at GL 4.1
  doesn't have the `glTexStorage` functions, although I heard
  that those functions are implemented but just not present in the
  core GL headers - which I'll need to investigate)
- I want to improve the internal 'lifetime tracking' for referenced
  resources (e.g. one resource object holding a reference to another
  object). Currently it's not possible to detect when such a referenced
  object has gone through an 'uninit/init' cycle because this keeps
  the same public handle while discarding and recreating backend
  3D API objects. Especially for view objects (which need to track
  their original resource object) it is important that views can
  detect when their referenced resource object is discarded (and
  I'm thinking about 'auto-managed' view objects which can recreate
  themselves on the fly when their resource object goes through
  uninit/init - no promises yet though).

More info on those planned updates are in the following planning
tickets:

- resource views: https://github.com/floooh/sokol/issues/1252
- better internal reference tracking: https://github.com/floooh/sokol/issues/1260
- glTexStorage vs glTexImage: https://github.com/floooh/sokol/issues/1263

...and that is all for today :)