---
layout: post
title: The sokol-gfx resource view update.
---

> NOTE: the WebGPU sample links point to the 'old' samples until the update is actually merged! The source code samples are uptodate though.

In a couple of days I will merge the next big (and breaking) sokol-gfx
update which adds resource view objects and in turn removes pre-baked
pass-attachment objects.

The update also requires to update sokol-shdc and recompile shaders.

The root PR is here: [https://github.com/floooh/sokol/pull/1287](https://github.com/floooh/sokol/pull/1287)

After merging the update I will spend a couple of weeks to take care of
pending issues and PRs before moving on to a followup [resource views update 2](https://github.com/floooh/sokol/issues/1302).

## What are resource view objects?

If you're familiar with D3D10 and later you'll feel right at home since
resource views are a fundamental concept in D3D, and sokol-gfx's concept
of resource views is closest to D3D11. Other 3D APIs either don't have
view objects at all (WebGL2 and GL before version 4.3), or only associate resource
views with texture data but not buffer data (GL >= 4.3, Metal and WebGPU).

Typically resource views have a number of different purposes in the various
3D-APIs:

- they specialize a parent resource object for a specific usage in shaders
  (for instance sampling the same image object as a texture versus using the
  image object as render target)
- they can reinterpret the data in a resource object (for instance to a
  different pixel format or image type)
- they can define a subset of the data in the resource object (for instance
  selecting a specific mipmap or range of mipmaps in a texture)

In sokol-gfx you can think of view objects mainly as specializations of an
`sg_image` or `sg_buffer` object for how the image or buffer is going to be accessed in
shaders:

- sampling a texture in a shader requires a **texture view**
- writing to a storage image in a compute shader requires a **storage image view**
- accessing a storage buffer in a shader requires a **storage buffer view**
- each render pass attachment type requires its own view object type:
    - **color-attachment views**
    - **resolve-attachment views**
    - **depth-stencil-attachment views**

Alternatively you can think of view objects as specializations of a resource
object for a specific bindings type (I was actually considering calling this new
object type `sg_binding`, but since 'view' is the more established term I went
with `sg_view` instead).

In sokol-gfx, resource view types are 'runtime flavours' of the same handle type
`sg_view`. This means that setting the wrong resource type on a bindslot won't
be a compilation error, but a runtime error in the sokol-gfx validation layer,
so please make sure to test your code in debug build mode from time to time.


## New unlocked features

This first sokol-gfx resource view update unlocks the following features:

- Storage buffer bindings can now have an offset. Binding storage buffers
  with offsets is mainly useful when the same buffer contains different
  types of items in different sections of the buffer, and processing
  those items in separate compute shaders - or if you only need to access a section
  of a buffer with a compute shader.
- Texture views can define a subset of the parent image by defining
  their own mipmap- and slice-ranges (not on WebGL, GLES3 or GL4.1 - e.g. macOS)
- Storage images are no longer 'compute pass attachments', but instead
  bound like regular textures in the `sg_apply_bindings()` call. This
  allows writing to many different storage images in the same compute pass
  (the number of simultaneously bound storage images is still very restricted
  though)
- Combinations of render pass attachment images are no longer 'pre-baked'
  into `sg_attachments` objects. Instead `sg_attachments` is now a
  transient struct like `sg_bindings`. This relaxes another
  'combinatorial explosion scenario' because rendering code longer needs
  to predict all possible render-pass attachment combinations upfront.

## Current restrictions and planned features

The following resource view features are planned for a followup 'resource view update 2':

- Reinterpret the pixel format and image type of image objects in a view object.
- Change the max number of per-shader-stage resource bindings of the same type
  from hardwired conservative limits to dynamic device limits exposed in the
  `sg_limits` struct (e.g. more than 4 storage image, 8 storage buffer or 16 texture
  bindings - instead try to push those limits closer to 32)

For more details about planned 'update 2' features see:

[https://github.com/floooh/sokol/issues/1302](https://github.com/floooh/sokol/issues/1302)

## High level overview of public API changes

- the `sg_attachments` object type and related functions have been removed
- a new object type `sg_view` has been added along with related functions
- `sg_features` gained a new flag `.gl_texture_views`, when this is false the GL backend doesn't
  have full texture view support (e.g. it's not possible to limit a view to a miplevel or slices
  subset)
- the `sg_attachments` name has been repurposed for a transient struct of render pass
  attachment views:
    ```c
    typedef struct sg_attachments {
        sg_view colors[SG_MAX_COLOR_ATTACHMENTS];
        sg_view resolves[SG_MAX_COLOR_ATTACHMENTS];
        sg_view depth_stencil;
    } sg_attachments;
    ```
- the `sg_bindings` struct now has a unified array for views instead of separate
  arrays for each 'shader resource type' (textures, storage images and storage buffers):
    ```c
    typedef struct sg_bindings {
        // ...
        sg_view views[SG_MAX_VIEW_BINDSLOTS];
        // ...
    } sg_bindings;
    ```
- the `sg_image_usage` struct now has more detailed usage flags for
  render pass attachments, and the `.storage_attachment` usage flag
  has been renamed to `.storage_image`:
    ```c
    typedef struct sg_image_usage {
        bool storage_image;
        bool color_attachment;
        bool resolve_attachment;
        bool depth_stencil_attachment;
        // ...
    } sg_image_usage;
    ```
- in `sg_image_desc` the items to directly inject backend-specific view
  objects have been removed:
    - `d3d11_shader_resource_view`
    - `wgpu_texture_view`
- in `sg_shader_desc`:
    - the internals of the `sg_shader_desc` struct to describe the shader
    binding interface has been changed to a unified array of `sg_shader_view_desc`
    structs:
        ```c
        typedef struct sg_shader_desc {
            // ...
            sg_shader_view views[SG_MAX_VIEW_BINDSLOTS];
            // ...
        } sg_shader_desc;
        ```
    - some renaming to better differentiate between '(storage) image and texture
      bindings', for instance 'image-sampler-pairs' are now called 'texture-sampler-pairs',
      since only texture bindings are 'sampled', but not storage-image bindings
- many new items in the `sg_frame_stats` struct, mostly not directly related
  to resource views, but filling some gaps


## Shader Authoring Changes

> TL;DR: When recompiling existing shaders you might get new errors about bindslot
collisions which need to be resolved by changing the `layout(binding=N)`
decorations.

When using sokol-shdc, the only change on the shader side is that textures,
storage buffers and storage images now share a common bindslot range, previously
each binding type had its slot range:

```glsl
@cs cs
layout(binding=0) uniform texture2D cs_inp_tex;
layout(binding=0, rgba8) uniform writeonly image2D cs_outp_tex;
// ...
@end
```

Note how in this (old) code-snippet the texture- and storage-image bindings use
the same bindslot 0 because previously textures and storage images had their own
bindslot space.

This code will now produce a 'bindslot collision error' when compiled with
sokol-shdc, because texture- and storage-image bindings now use the same bindslot
space, so bindings for texture-, storage-buffer- and storage-image-bindings across
all shader stages need to be fixed to not collide:

```glsl
@cs cs
layout(binding=0) uniform texture2D cs_inp_tex;
layout(binding=1, rgba8) uniform writeonly image2D cs_outp_tex;
// ...
@end
```
This bindslot fixup is the only change required on the shader side.

## Working with Texture Views

Sample code:
- **texcube-sapp** (simple textured rendering): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/texcube-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/texcube-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/texcube-sapp-ui.html)
- **dyntex-sapp** (CPU-update dynamic texture): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/dyntex-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/dyntex-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/dyntex-sapp-ui.html)

Let's say a shader defines a texture binding at slot 3:

```glsl
layout(binding=3) uniform texture2D tex;
```

To 'populate' this bindslot on the CPU side you need two objects now: an image
object, and a texture view on the image object:

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 4,
    .height = 4,
    .data.subimage[0][0] = ...,
});
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture = { .image = img },
});
```

Since this is C you can also chain the designated initializers which looks a bit more compact
(unfortunately this isn't supported in most other languages):

```c
sg_view tex_view = sg_make_view(&(sg_view_desc){ .texture.image = img });
```

The `sg_apply_bindings()` call now has an array of `sg_view` handles instead
of separate arrays for images and storage buffers:
```c
sg_apply_bindings(&(sg_bindings){
    .vertex_buffers[0] = ...,
    .views[VIEW_tex] = tex_view,
    .samplers[SMP_smp] = ...,
});
```

Since the texture binding was defined as `layout(binding=3)` it's also
safe to just use the bind slot index directly instead of the code-generated
constant:

```c
sg_apply_bindings(&(sg_bindings){
    .vertex_buffers[0] = ...,
    .views[3] = tex_view,
    .samplers[SMP_smp] = ...,
});
```

In many situations you only need the view handle and don't need the
separate image handle, this means you can nest the `sg_make_image()`
inside the `sg_make_view()` call:

```c
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture.image = sg_make_image(&(sg_image_view){
        .width = 4,
        .height = 4,
        .data.subimage[0][0] = ...,
    }),
});
```

If you need the image handle later you can extract it from the
view object via `sg_query_view_image()`:
```c
sg_image img = sg_query_view_image(tex_view);
```

Texture views can select a subrange of mipmaps and slices of their parent
image (not supported on WebGL2, GLES3 or GL4.1):
```c
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture = {
        .image = img,
        .mip_levels = { .base = 1, .count = 3 },
        .slices = { .base = 5, .count = 2 },
    },
});
```

If `.count` is left at default-zero it means 'all remaining mipmaps or slices'.
For instance this will only skip the most detailed mipmap but keep the
remaining mipmap chain in place:

```c
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture = {
        .image = img,
        .mip_levels = { .base = 1 },
    },
});
```

## View vs parent resource lifetime considerations

Before moving on to the other view types, a little interlude about
lifetimes and resource states:

If you're coming from 3D APIs with ref-counted lifetime management like D3D, WebGPU
or Metal you might be tempted to 'release' a view's parent resource object right after creating
its view object if the image object handle isn't needed anymore:

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 4,
    .height = 4,
    .data.subimage[0][0] = ...,
});
sg_view tex_view = sg_make_view(&(sg_view){ .texture.image = img });
sg_destroy_image(img);
```

In sokol-gfx lifetimes are explicit, if you pull the rug under a view
like this nothing catastrophic will happen (e.g. no crashes or hard
validation layers errors), but rendering operations involving such 'dangling views'
will be silently skipped (this is basically the same behavior as before
when trying to render with images or buffers in a non-valid resource state).

Another slightly counter-intuitive behavior might be that a view object
remains in valid resource state despite its parent resource being destroyed, e.g.
following the above example code:

```c
// get the destroyed image's resource state
if (sg_query_image_state(img) == SG_RESOURCESTATE_INVALID) {
    // if-branch taken, since the image had been destroyed
    // ...
}
// get the image's texture view resource state
if (sg_query_view_state(tex_view) == SG_RESOURCESTATE_VALID) {
    // if-branch *also* taken!
    // ...
}
```

I went a bit back and forth on this decision but I think the behavior makes
sense from the perspective that all resource state changes in sokol-gfx
are explicit (e.g. there are no 'automatic' state changes as
a side effect of a 'remote' state change of another object, instead all resource state changes
are directly caused by a function call on that resource object). The same
has always been true for pipelines and their shader object, just not
specifically documented.

If you want to check whether a view is 'renderable' you can use
the following shortcut:

```c
if (sg_query_image_state(sg_query_view_image(tex_view)) == SG_RESOURCESTATE_VALID) {
    // the view is 'renderable'
}

// or for storage buffer views:
if (sg_query_buffer_state(sg_query_view_buffer(sbuf_view)) == SG_RESOURCESTATE_VALID) {
    // the view is 'renderable'
}
```
This works because no matter what state the view object is in (or even exists),
`sq_query_view_image()` will either return an image handle or an invalid handle
and both can be passed into `sg_query_image_state()`. An invalid image handle
will return `SG_RESOURCESTATE_INVALID` while a valid image handle will return
the actual `SG_RESOURCESTATE_*` of the image object.

## Tracking uninit => init cycles

If the parent resource goes through a 'destroy => make' or 'uninit => init' cycle,
all views which had been created from this parent resource must also be
re-initialized, otherwise rendering operations involving such 'dangling views'
will silently be skipped.

A common pattern for this situation is to use the 'uninit => init' calls instead
of 'destroy => make' because the handles will remain valid (e.g. you don't need
to distribute new object handles into all corners of your code base):

```c
// first uninit/init the parent image with new params:
sg_unit_image(img);
sg_init_image(img, &(sg_image_desc){ ... });
// then 'cycle' the image's view objects
sg_uninit_view(tex_view);
sg_init_view(tex_view, &(sg_view_desc){ .texture.image = img });
```

I was at first considering to add a 'managed mode' for views which would track
the state of their parent resource and automatically go through an uninit/init
cycle when needed, but this just didn't fit into the sokol philosophy of explicit
lifetimes and resource states, and having this one special case for view objects
caused more confusion which wasn't worth the small gain in convenience (this
decision also wasn't purely based on gut feeling since I actually *had*
implemented the 'managed mode' already but then kicked it out again after
actually starting to port the sokol sample code over - it just didn't 'feel right').

When porting existing code over to resource view object, don't forget
that you need to destroy at least two objects now for complete cleanup
(views *and* their parent resource).

The order in which you destroy the views and parent resources doesn't
matter, this:
```c
sg_destroy_view(view);
sg_destroy_image(img);
```
...works just as well as this:
```c
sg_destroy_image(img);
sg_destroy_view(view);
```

**BUT BE AWARE OF THIS TRAP:**
```c
sg_destroy_view(view);
sg_destroy_image(sg_query_view_image(view));
```

Since the view is already destroyed, `sg_query_view_image()` will return the invalid
handle, and passing the invalid handle into `sg_destroy_image()` is a silent no-op
(e.g. your image will leak).

...this is actually a nice example of how convenience in one situation (calling
`sg_query_view_image(view)` and `sg_destroy_image()` with an invalid handle
being a silent no-op) can cause trouble in other situations. I'll need to think
about whether this should at least be logged as an error instead.

## Working with render pass attachment views

Sample code:

- **offscreen-sapp** (simple offscreen rendering): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/offscreen-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/offscreen-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/offscreen-sapp-ui.html)
- **offscreen-msaa-sapp** (multi-sampled offscreen rendering): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/offscreen-msaa-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/offscreen-msaa-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/offscreen-msaa-sapp-ui.html)
- **mrt-sapp** (multiple-render-target, multi-sampled offscreen rendering): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/mrt-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/mrt-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/mrt-sap-ui.html)
- **mrt-pixelformats-sapp** (multiple render target rendering with different pixel formats): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/mrt-pixelformats-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/mrt-pixelformats-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/mrt-pixelformats-sapp-ui.html)
- **shadows-sapp** (shadow-mapping with regular shadow map texture): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/shadows-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/shadows-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/shadows-sapp-ui.html)
- **shadows-depthtex-sapp** (shadow-mapping with a depth-buffer texture): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/shadows-depthtex-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/shadows-depthtex-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/shadows-depthtex-sapp-ui.html)
- **miprender-sapp** (render into mipmaps): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/miprender-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/miprender-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/miprender-sapp-ui.html)
- **layerrender-sapp** (render into array slice): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/layerrender-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/layerrender-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/layerrender-sapp-ui.html)

In the previous sokol-gfx version, when doing offscreen rendering into an image object
a 'pre-baked' attachments object had to be created which was then passed into `sg_begin_pass()`:

E.g. old code:
```c
// create a color and depth-buffer image for offscreen rendering
sg_image color_img = sg_make_image(&(sg_image_desc){
    .usage = { .render_attachment = true },
    // ...
});
sg_image depth_img = sg_make_image(&(sg_image_desc){
    .usage = { .render_attachment = true },
    // ...
});

// create an attachments object from those images...
sg_attachments atts = sg_make_attachments(&(sg_attachments_desc){
    .colors[0].image = color_img,
    .depth_stencil.image = depth_img,
});

// ... in the render loop for the offscreen render pass:
sg_begin_pass(&(sg_pass){ .attachments = atts });
// ...
sg_end_pass();

// ... and in the swapchain pass, bind the color image as texture:
sg_apply_bindings(&(sg_bindings){
    // ...
    .images[TEX_tex] = color_img,
    // ...
});
```

Now, instead of creating a pre-baked attachments object, separate 'attachment-view'
objects are created upfront, but their combined use for rendering is no longer
pre-baked but defined on-the-fly in the `sg_begin_pass()` call, much like
bindings in the `sg_apply_bindings()` call:

```c
// create color- and depth-buffer images
// NOTE the more detailed usage flags
sg_image color_img = sg_make_image(&(sg_image_desc){
    .usage = { .color_attachment = true },
    // ...
});
sg_image depth_img = sg_make_image(&(sg_image_desc){
    .usage = { .depth_stencil_attachment = true },
    // ...
});

// create color- and depth-stencil attachment views
sg_view color_att_view = sg_make_view(&(sg_view_desc){
    .color_attachment.image = color_img,
});
sg_view depth_att_view = sg_make_view(&(sg_view_desc){
    .depth_stencil_attachment.image = depth_img,
});

// since the color-attachment image is also sampled as texture,
// we'll also need a texture view:
sg_view color_tex_view = sg_make_view(&(sg_view_desc){
    .texture.image = color_img,
});

// later in the offscreen render pass, the attachment views
// are passed directly into sg_begin_pass:
sg_begin_pass(&(sg_pass_desc){
    .attachments = {
        .colors[0] = color_att_view,
        .depth_stencil = depth_att_view,
    },
});
// ...
sg_end_pass();

// and in the swapchain pass, the texture view is bound
// to sample the offscreen-rendered image as texture:
sg_apply_bindings(&(sg_bindings){
    // ...
    .views[VIEW_tex] = color_tex_view,
    // ...
});
```

## Working with storage image views

Samples:

- **write-storageimage-sapp** (write into storage image with compute shader): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/write-storageimage-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/write-storageimage-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/write-storageimage-sapp-ui.html)
- **imageblur-sapp** (image blurring with compute shaders): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/imageblur-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/imageblur-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/imageblur-sapp.html)

Storage image bindings are no longer defined as compute-pass attachments in `sg_begin_pass()`, but instead
like regular texture- or storage-buffer-bindings in `sg_apply_bindings()`.

```c
// first create an image object with storage-image usage:
sg_image img = sg_make_image(&(sg_image_desc){
    .usage = { .storage_image = true },
    // ...
});

// to write to the image with a compute shader, a storage image view is needed:
sg_view simg_view = sg_make_view(&(sg_view_desc){
    .storage_image = {
        .image = img,
        .mip_level = ...,   // optional: select a specific miplevel
        .slice = ...,       // optional: select a specific slice
    },
});

// ...and to sample that same image as a texture for rendering, a texture view is needed:
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture.image = img,
});

// storage image views are now applied as regular bindings in a compute pass:
sg_begin_pass(&(sg_pass){ .compute = true });
// ...
sg_apply_bindings(&(sg_bindings){
    .views[VIEW_simg] = simg_view,
})
sg_dispatch(...);
sg_end_pass();

// and to use the compute-shader-updated image as a texture in a render pass,
// bind the texture view as usual:
sg_begin_pass(...);
// ...
sg_apply_bindings(&(sg_bindings){
    // ...
    .views[VIEW_tex] = tex_view,
    .samplers[SMP_smp] = smp,
});
sg_draw(...);
sg_end_pass();
```

## Working with storage buffer views

Samples:

- **vertexpull-sapp** (vertex pulling from storage buffer): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/vertexpull-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/vertexpull-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/vertexpull-sapp-ui.html)
- **sbuftex-sapp** (access storage buffer in fragment shader): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/sbuftex-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/sbuftex-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/sbuftex-sapp-ui.html)
- **instancing-compute-sapp** (update instancing data with compute shader): [C code](https://github.com/floooh/sokol-samples/blob/master/sapp/instancing-compute-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/master/sapp/instancing-compute-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/instancing-compute-sapp-ui.html)
- **sbufoffset-sapp** (demonstrate storage buffer bindings with offset): [C code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/sbufoffset-sapp.c), [GLSL code](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/sapp/sbufoffset-sapp.glsl), [WebGPU sample](https://floooh.github.io/sokol-webgpu/sbufoffset-sapp-ui.html)


To bind a buffer object as storage buffer for vertex-pulling or compute-shader access you now need a storage-buffer-view object:

```c
// create a buffer with storage-buffer usage:
sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .usage = { .storage_buffer = true },
    // ...
});

// create a storage buffer view
sg_view sbuf_view = sg_make_view(&(sg_view_desc){
    .storage_buffer = {
        .buffer = buf,
        .offset = ...,  // optional 256-byte aligned offset
    }
});

// ...later in a render- or compute-pass bind the storage buffer view:
sg_apply_bindings(&(sg_bindings){
    .views[VIEW_ssbo] = sbuf_view,
});
```

The 256-byte-alignment restriction for the offset is a bit unfortunate, since
vertex-buffer and index-buffer bind offsets don't have that restriction. The
alignment restriction is coming in via WebGPU which on some Android devices
requires this 256 byte alignment, but the only realistic lower choice would be
64 bytes which frankly isn't that much better
(https://vulkan.gpuinfo.org/displaydevicelimit.php?platform=android&name=minStorageBufferOffsetAlignment)
and would still exclude about 8 percent of Android devices which is quite a lot.

## When not using sokol-shdc...

Samples:

- for [D3D11](https://github.com/floooh/sokol-samples/tree/issue1252/resource_views/d3d11)
- for [Metal](https://github.com/floooh/sokol-samples/tree/issue1252/resource_views/metal)
- for [desktop GL](https://github.com/floooh/sokol-samples/tree/issue1252/resource_views/glfw)
- for [WebGL2](https://github.com/floooh/sokol-samples/tree/issue1252/resource_views/html5)
- for [WebGPU](https://github.com/floooh/sokol-samples/tree/issue1252/resource_views/wgpu)

Some tweaks on the manually populated `sg_shader_desc` structs are needed when not
using sokol-shdc:

- The separate bindslot reflection arrays for images, storage-buffers and storage-images
  have been unified into a `views[]` array which mirrors the `views[]` array in the
  `sg_bindings` struct. The actual reflection information in each view bindslot
  has remained the same though.
- The `.image_sampler_pair` array has been renamed to `.texture_sampler_array`, and
  the struct member `.image_slot` has been renamed to `.view_slot`.

Example from the [wgpu/mrt_wgpu.c sample](https://github.com/floooh/sokol-samples/blob/issue1252/resource_views/wgpu/mrt-wgpu.c):

```c
sg_shader fsq_shd = sg_make_shader(&(sg_shader_desc){
    // ...
    .views = {
        [0].texture = { .stage = SG_SHADERSTAGE_FRAGMENT, .wgsl_group1_binding_n = 0 },
        [1].texture = { .stage = SG_SHADERSTAGE_FRAGMENT, .wgsl_group1_binding_n = 1 },
        [2].texture = { .stage = SG_SHADERSTAGE_FRAGMENT, .wgsl_group1_binding_n = 2 },
    },
    .samplers = {
        [0] = { .stage = SG_SHADERSTAGE_FRAGMENT, .wgsl_group1_binding_n = 3 },
    },
    .texture_sampler_pairs = {
        [0] = { .stage = SG_SHADERSTAGE_FRAGMENT, .view_slot = 0, .sampler_slot = 0 },
        [1] = { .stage = SG_SHADERSTAGE_FRAGMENT, .view_slot = 1, .sampler_slot = 0 },
        [2] = { .stage = SG_SHADERSTAGE_FRAGMENT, .view_slot = 2, .sampler_slot = 0 },
    },
});
```

Shader code changes are only needed on WebGPU when using storage images. Those have
moved from `@group(2)` into `@group(1)` (this is because storage images are no longer
special compute-pass-attachments, but regular bindings just like texture- and
storage-buffer bindings).


## Q & A

### Why no vertex- and index-buffer views

I had actually implemented vertex- and index-buffer views at first because it
would have reduced the size of `sg_bindings` by 36 bytes (32 bytes vertex-buffer-offsets and 4 bytes
index-buffer-offset). In the end I rolled that change back since none of the
backend 3D APIs require creating view objects for binding vertex- and index-buffers, but
some rendering scenarios (like writing a renderer backend for Dear ImGui) heavily
depend on dynamic offsets for vertex- and index-data.

I might come back to that idea once additional drawing functions with base-offsets
are added (which is planned for the 'not-too-distant future'). Also adding
a D3D12 backend would require adding view objects for vertex- and index-buffers,
since D3D12 has removed the ability to bind vertex- and index-buffers directly
with a dynamic offset (at least that's what I'm seeing in the D3D12 docs).

### Why no 'texture' field in sg_image_usage to indicate that texture views may be created for an image object?

Simply because creating a texture view is always supported for image objects, so
that flag could be implicitly hardwired to true anyway (with one 'legacy edge
case': WebGL2 and GL4.1 not supporting binding multi-sampled images as
textures). In that edge-case, an explicit `.usage.texture` flag would allow to fail already at
image object creation instead of failing to create a texture view on a
multi-sampled image object, but since this is such a minor detail which only affects
'legacy APIs' (WebGL2 and GL 4.1) that I didn't think adding an explicit texture
usage flag was worth it.

### What's up with SG_MAX_VIEW_BINDSLOTS being this odd 28 instead of some 2^N value?

That way the `sg_bindings` struct is a nice round 256 bytes (64 bytes for vertex
buffer handles and offsets, 8 bytes for index buffer and offset, 112 bytes for
view handles, 64 bytes for sampler handles plus 2*4 bytes for the start and end
canaries).

16 separate samplers might be overkill, so I might tweak the number of views vs
samplers a bit in the 'resource view update 2'.