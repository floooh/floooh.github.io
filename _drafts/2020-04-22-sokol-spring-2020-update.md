---
layout: post
title: "Sokol headers: spring 2020 update"
---

I'm a few days away from merging the last 2.5 months of work on sokol_gfx.h
and sokol_app.h back into the master branch. The bulk of this is an
experimental WebGPU backend for sokol_gfx.h ("experimental" because the
WebGPU API is still in flux). Since the WebGPU backend required one
small change in the public API of sokol_gfx.h I took the chance and
did another small breaking change which I had in the back of my head for quite a while.

# Overview of changes

First a quick overview of what's new and what has changed. Each change
will then be described in detail.

If you're interested in the nitty-gritty details you can review
this pull request:

https://github.com/floooh/sokol/pull/259

## Changes in sokol_gfx.h:

- an experimental WebGPU backend
- a new enum type ```sg_sampler_type```
- a new item ```sampler_type``` in struct ```sg_shader_image_desc```
- new items and a new internal layout in struct ```sg_desc```

Only the last point is an actual breaking change, but the required
source code updates are minimal (more on that further below).

## Changes in sokol_app.h:

- a new enum ```sapp_pixel_format``` which is a small value-compatible subset
  of ```sg_pixel_format``` from sokol_gfx.h for the renderable
  default-framebuffer
pixel formats supported on all platforms
- new functions to query default-framebuffer attributes:
    - ```uint32_t sapp_color_format(void)```
    - ```uint32_t sapp_depth_format(void)```
    - ```int sapp_sample_count(void)```
- WebGPU device- and swapchain-setup code (so far only in the
emscripten code-path)
- new functions ```sapp_wgpu_*``` to query WebGPU device and swapchain object handles

## Other changes:

- a new header ```sokol_glue.h``` which will contain a slowly growing
set of "header interop helper functions" which are useful when 
specific combinations of sokol headers are used together
- the ```sokol-shdc``` shader cross-compiler has a new output option
for WebGPU which currently produces SPIRV-blobs (later this will be changed
to the new WSL shading language)

# The changes in detail

## Experimental WebGPU backend

This really deserves its own blog post so I'll just give a very broad
overview here for now:

The WebGPU backend is selected with the new ```SOKOL_WGPU``` preprocessor
define both in sokol_gfx.h and sokol_app.h. 

It's currently called "experimental" because the WebGPU API itself still
changes a lot, especially in the area of resource management and
shaders.

The implementation is based on the current "Google WebGPU flavour", using
the WebGPU C-API available both for [Google native implementation](https://dawn.googlesource.com/dawn)
and the [webgpu.h header plus JS shim](https://github.com/emscripten-core/emscripten/blob/master/system/include/webgpu/webgpu.h), and SPIRV binary blobs for shaders.

Just as with WebGL vs GL, the WebGPU backend in sokol_gfx.h doesn't care about
whether it's running in a native environment through a native WebGPU
implementation (e.g. Google Dawn), or in a web environment through emscripten.

The same isn't the true yet for sokol_app.h though: currently only the emscripten code
path has the required code for setting up the WebGPU device and swapchain. But
the plan for sokol_app.h is definitely to also add support for native WebGPU
libraries, one reason for this is because it dramatically simplifies the
debugging situation (because debugging native code is a lot easier currently
than debugging WASM binaries).

Device and swapchain setup for WebGPU outside the emscripten backend depends on the
availability of C-APIs to do this which is (currently?) out of the scope of
the WebGPU C-API. Hopefully the different native WebGPU implementations will also
agree on a common cross-platform C-API for the "platform bindings" (if not
that shouldn't be a big deal either though, because it's not that much code).

## The sg_sampler_type enum

This is a new enumeration type in the public API of sokol_gfx.h which had to be
added for WebGPU. The background is that texture bindings in WebGPU need
to know whether the texture will be sampled as float, signed integer or
unsigned integer in the vertex- or fragment-shader.

The corresponding WebGPU enum is [here](https://github.com/emscripten-core/emscripten/blob/4d4de90be481a12a69ea25b21d28d0cbc675e650/system/include/webgpu/webgpu.h#L274-L279).

In sokol_gfx.h, the sampler type has been added to the ```sg_shader_image_desc```
as ```.sampler_type```:

```c
typedef struct sg_shader_image_desc {
    const char* name;
    sg_image_type type;
    sg_sampler_type sampler_type;
} sg_shader_image_desc;
```

> NOTE: I was considering to rename the existing ```type``` member to 
```image_type``` but decided against that for now for the sake of not
introducing too many breaking changes at the same time. Might be an
option for later though.

The simple rules-of-thumb for this new item are:

- if you don't use the WebGPU backend, you can safely ignore it
- if you use sokol-shdc you can also ignore it because it will be automitcally
filled into the code-generated sg_shader_desc struct from the shader's
reflection information
- otherwise, with GLSL as source language as example, the GLSL sampler type
must correspond with the sokol_gfx.h sg_sampler_type enum values like this:
    - sampler*: SG_SAMPLERTYPE_FLOAT
    - isampler*: SG_SAMPLERTYPE_INT
    - usampler*: SG_SAMPLERTYPE_UINT

Unfortunately there isn't really a way to check whether the sampler types
match the underlying shader samplers in the sokol_gfx.h validation layer,
since this would require access to shader reflection information in all
backends. So instead of an "early validation" in sokol_gfx.h, a mismatch
can only be caught in the underlying WebGPU validation layer.

The actual reason why this new "resource binding attribute" even exists in WebGPU
is to do an early validation check at "setup time" instead of having to delay
this until "draw time".

## New layout for the sg_desc struct

This change has the biggest impact on existing code because it is a breaking
change. I grew a little bit more unhappy with the sg_desc struct with each new
rendering backend added to sokol_gfx.h, and I felt that now a tipping
point was reached where some changes were needed.

Long story short, this is what the new ```sg_desc``` struct looks like
when the initialization is "written out" in C99 designated initialization syntax:

```c
sg_desc desc = {
    .buffer_pool_size = ...,
    .image_pool_size = ...,
    .shader_pool_size = ...,
    .pipeline_pool_size = ...,
    .pass_pool_size = ...,
    .context_pool_size = ...,
    .uniform_buffer_size = ...,
    .staging_buffer_size = ...,
    .sampler_cache_size = ...,

    .context = {
        .color_format = ...,
        .depth_format = ...,
        .sample_count = ...,
        .gl = {
            .force_gles1 = ...,
        },
        .metal = {
            .device = ...,
            .renderpass_descriptor_cb = ...,
            .drawable_cb = ...,
        },
        .d3d11 = {
            .device = ...,
            .device_context = ...,
            .render_target_view_cb = ...,
            .depth_stencil_view_cb = ...,
        },
        .wgpu = {
            .device = ...,
            .render_view_cb = ...,
            .resolve_view_cb = ...,
            .depth_stencil_view_cb = ...,
        }
    }
};
```

There's a couple of new items, and other items have been shuffled around a bit,
and as you can see, the information needed to "bind" sokol_gfx.h to a specific
backend 3D-API "context" is getting quite lengthy.

Providing all the information needed for a cross-platform program which needs
to render on all the supported 3D backends is starting to take up a significant
portion of a "Hello Triangle" line count (this is where the new ```sokol_glue.h```
header is coming in, but I'm getting ahead of myself).

First let's look at the new and changes items:

### sg_desc.uniform_buffer_size

Formerly this was called ```mtl_global_uniform_buffer_size```, 
was specific to the Metal backend and described the size of the "per frame uniform buffer",
e.g. the buffer that needs to hold all uniform data updated in a single frame
via calls to ```sg_update_uniforms()```. The new WebGPU backend has a similar uniform
update strategy as the Metal backend, so this is a "general" setup parameter now
(it's only used in the Metal and WebGPU backend though).

### sg_desc.sampler_cache_size

This was also a Metal-specific parameter before
called ```mtl_sampler_cache_size```. It describes the *number* of unique
entries in an internal cache for texture sampler objects. The WebGPU backend
also has such a sampler cache, so the setup parameter has been generalized.
But again, this parameter is only used in the Metal and WebGPU backend.

### sg_desc.staging_buffer_size

Currently used only in the WebGPU backend, this
configuration parameter describes the size in bytes of a "per frame staging buffer" 
for dynamic data uploads from CPU-visible to GPU-visible memory. The staging
buffer size must be big enough to hold all the dynamically updated data via
```sg_update_buffer()```, ```sg_append_buffer()``` and ```sg_update_image()```
happening in a single frame. This part of the WebGPU API is still heavily in 
flux though, so maybe such a user-side staging buffer won't actually be necessary
in the future, so maybe this configuration parameter will disappear again.

### sg_desc.context

All the backend-specific information required to "bind" sokol_gfx.h to a 3D-API
context (e.g. rendering device and swapchain) has been moved into a nested
structure of the new type ```sg_context_desc```, and in there each
backend-portion (GL, Metal, D3D11 and WebGPU) has gotten its own nested
structure as well. 

Those nested structs have a specific reason: they allow to initialize the related
items in the nested struct with a single assignment inside a designated initialization
block.

[to be continued]
