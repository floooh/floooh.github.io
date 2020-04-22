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

This is a new enumeration in the public API of sokol_gfx.h which had to be
added for WebGPU. The background is that texture bindings in WebGPU need
to know whether the texture will be sampled as float, signed integer or
unsigned integer in the shader.

The corresponding WebGPU enum is [here](https://github.com/emscripten-core/emscripten/blob/4d4de90be481a12a69ea25b21d28d0cbc675e650/system/include/webgpu/webgpu.h#L274-L279).

In sokol_gfx.h, the sampler type has been added to the ```sg_shader_image_desc```
as ```sampler_type```:

```c
typedef struct sg_shader_image_desc {
    const char* name;
    sg_image_type type;
    sg_sampler_type sampler_type;
} sg_shader_image_desc;
```

> NOTE: I was considering to rename the existing ```type``` member to 
```image_type``` but decided against that for now for the sake of not
introducing too many breaking changes at the same time.

```sg_shader_image_desc``` is a nested struct in ```sg_shader_desc``` for
describing texture sampling "slots" in the vertex- or fragment-shader-stage,
for instance:

```c
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    .vs = {
        .source = "..."
    },
    .fs = {
        .images = {
            [0] = {
                .name = "tex",
                .type = SG_IMAGETYPE_2D,
                .sampler_type = SG_SAMPLERTYPE_FLOAT // NEW!
            }
        }
        .source = "..."
    }
});
```

The simple rules-of-thumb for this new item are:

- if you don't care about WebGPU, you can ignore it
- if you use sokol-shdc you can also ignore it because it will be automitcally
filled into the code-generated sg_shader_desc struct from the shader's
reflection information
- otherwise, with GLSL as source language, the GLSL sampler type
corresponds with the sokol_gfx.h sg_sampler_type enum values like this:
    - sampler*: SG_SAMPLERTYPE_FLOAT
    - isampler*: SG_SAMPLERTYPE_INT
    - usampler*: SG_SAMPLERTYPE_UINT

Unfortunately there isn't really a way to check whether the sampler types
match the underlying shader samplers in the sokol_gfx.h validation layer,
since this would require access to shader reflection information in all
backends. So instead of an "early validation" in sokol_gfx.h, this sort
of error will be caught by the underlying WebGPU validation layer.

