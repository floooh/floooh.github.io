---
layout: post
title: The sokol-gfx Compute Milestone 2 update
---

In a couple of days I will merge the next breaking sokol_gfx.h update (aka the `compute-ms2`
update) which makes working with buffer objects a bit more flexible and will allow
compute shaders to write to texture objects via 'compute pass attachments'.

For the latter feature you'll also need to update sokol-shdc to the latest version.

> NOTE: all sample URLs require a WebGPU capable browser and will only
> be valid after the merge.

The implementation ticket is here, and this also has links to all related
pull requests: [https://github.com/floooh/sokol/issues/1244](https://github.com/floooh/sokol/issues/1244)

### An important behaviour change for immutable buffer objects

The initial 'compute shader' update allows to create immutable buffers without
initial data and guaranteed that the buffer content would be zero-initialized.
On some backend APIs this required a temporary memory allocation of the buffer
size which obviously wasn't great.

This guaranteed zero-initialization has been rolled back now and the rules
for creating immutable buffer objects have been changed like this:

- when creating an immutable non-storage-buffer object (e.g. the buffer cannot
  be written to with a compute shader), initial data *must* be provided
- when creating an immutable storage-buffer object, no initial data needs to
  provided, but the buffer content will be 'undefined'

In practice this means that when you use a compute shader to initialize
storage buffer content you can no longer rely on the buffer content being
zero-initialized.

### Multi-purpose buffer objects

It's now possible to bind the same buffer object to different bind points (e.g.
bind the same buffer as vertex buffer, index buffer of storage buffer). This
means the following scenarios are now enabled:

- It's possible to stash vertices and indices into the same buffer
  (with the exception of WebGL2 where this is explicitly disallowed)
- It's now possible to use a compute shader to write data to a buffer, and
  then bind this buffer as vertex- or index-buffer.

To achieve this, the `sg_buffer_desc` has been changed to merge the previous
buffer type and buffer usage enums into a new `sg_buffer_usage` struct which is a
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

Note that 'multi-purpose buffer usage' is not allowed on WebGL2 (which is
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

The following sample uses a combined vertex/index buffer:

- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.glsl](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/vertexindexbuffer-sapp.glsl)
- WASM: [https://floooh.github.io/sokol-webgpu/vertexindexbuffer-sapp-ui.html](https://floooh.github.io/sokol-webgpu/vertexindexbuffer-sapp-ui.html)

The `instancing-compute-sapp` has been updated to bind the compute-shader-updated
storage buffer as vertex buffer with hardware instancing:

- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.glsl](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/instancing-compute-sapp.glsl)
- WASM: [https://floooh.github.io/sokol-webgpu/instancing-compute-sapp-ui.html](https://floooh.github.io/sokol-webgpu/instancing-compute-sapp-ui.html)

There is no sample yet which uses a compute shader to write index data.

### Breaking changes when creating image objects

Similar to the above `sg_buffer_desc` change, usage hints in the `sg_image_desc` struct
are now provided through a new `sg_image_desc` struct looking like this:

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

...and creating a dynamic image with stream update behaviour now looks like this:

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

It's now possible to use compute shaders to write to texture objects. The way this is currently
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

On the CPU side, create a storage image with 'storage attachment usage':

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

In the frame loop, start a compute pass and provide the attachments object,
apply the compute pipeline and uniform data, and finally call `sg_dispatch()`:

```c
sg_begin_pass(&(sg_pass){ .compute = true, .attachments = atts });
sg_apply_pipeline(pip);
sg_apply_uniforms(UB_cs_params, &SG_RANGE(cs_params));
sg_dispatch(WIDTH / 16, HEIGHT / 16, 1);
sg_end_pass();
```

...after the compute pass the image object can be used as a texture binding in a followup render pass.

Find the complete sample here:

- WASM: [https://floooh.github.io/sokol-webgpu/write-storageimage-sapp.html](https://floooh.github.io/sokol-webgpu/write-storageimage-sapp.html)
- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.glsl](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/write-storageimage-sapp.glsl)

...and a more advanced example:

- WASM: [https://floooh.github.io/sokol-webgpu/imageblur-sapp.html](https://floooh.github.io/sokol-webgpu/imageblur-sapp.html)
- C code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c)
- GLSL code: [https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c](https://github.com/floooh/sokol-samples/blob/issue1244/compute-ms2/sapp/imageblur-sapp.c)

### Detailed change list

### What's next



TODO outline:

- breaking changes:
    - sg_buffer_desc.usage
    - sg_image_desc.usage
- multi-purpose buffers:
    - combined vertex/index buffers
    - storage buffers as vertex/index buffers
- storage images/attachments overview
    - storage image creation
    - storage attachment creation
- shaders with storage images
    - compute shader examples
    - sg_shader_desc.storage_images[]
- compute passes with storage attachments
- roadmap:
    - gl texture creation update (glTexStorage)
    - safer internal references + init count
    - resource view objects
