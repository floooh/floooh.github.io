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
whether it's running in a native environment via a native WebGPU
implementation library (e.g. Google Dawn), or in a web environment via a
JS shim provided by the emscripten SDK.

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
in the future and this configuration parameter might disappear again.

### sg_desc.context

All the backend-specific information required to "bind" sokol_gfx.h to a 3D-API
context (e.g. rendering device and swapchain) has been moved into a nested
structure of the new type ```sg_context_desc```, and in there each
backend-portion (GL, Metal, D3D11 and WebGPU) has gotten its own nested
structure as well. 

Those nested structs have a specific reason: they allow to initialize the related
items in the nested struct with a single assignment inside a C99 designated initialization
block.

More on this below where I'm introducing the new sokol_glue.h header.

The nested context struct has 3 new members to describe the pixel formats
and MSAA sample count of the default framebuffer (which must be created
outside of sokol-gfx):

```c
sg_desc desc = {
    .context = {
        .color_format = ..., // color buffer format as sg_pixel_format
        .depth_format = ..., // depth stencil  format as sg_pixel_format
        .sample_count = ..., // MSAA sample count
    }
};
```

These new configuration values fix some hardwired assumptions in the
sokol_gfx.h API and make it a bit more convenient to create pipeline state
objects that are compatible with the default frame buffer, because the default
values for the pipeline state's pixel color- and depth-format, and the
sample count are no longer hardwired, but taken from the one-time values
provided in sg_desc.

Example:

If the default framebuffer uses MSAA antialiased rendering, previously you had to provide
the matching sample count when creating a pipeline state object, because sokol_gfx.h
didn't actually know that the default framebuffer is multisampled:

```c
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    ...
    .rasterizer.sample_count = 4,
    ...
});
```

Now that the default framebuffer's sample count is provided upfront in sg_setup(),
this is used as the default value for pipeline state object creation, and
the sample_count can be left zero-initialized when creating an sg_pipeline
object which used for rendering to the default frame buffer:

```c
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    ...
});
```

The other improvement is that there's now a new validation check which makes sure
that the sample count of a pipeline state object matches the default-framebuffer's
sample count.

There's a somewhat unexpected side effect regarding this new default-sample-count
behaviour:

When creating a render target image, the pixel format and sample count
will *also* be initialized from the default-framebuffer's color format and
sample count, so (by default) offscreen render targets use the same
"pixel configuration" as the default frame buffer, which doesn't
really make sense because the two aren't related.

After some back and forth I decided on this behaviour so that a
default-initialized render target image will always be compatible with a
default-initialized pipeline state object.

Admittedly, that's almost a bit too much 'under-the-hood-magic' for my taste,
but I think in this specific situation it's justified because the whole idea
of having 'useful defaults' to reduce line count was a central design goal
of sokol_gfx.h from the start.

You'll just have to keep in mind that if your default-framebuffer is 
antialiased, than all your render targets will also (by default!) be 
antialiased, unless you explicitely create the render targets (and their
matching pipeline state objects) with a sample_count of 1.

On to the next topic:

## The sokol_glue.h header

As described above, the main reason for adding a nested structure to 
sg_desc to hold all the 3D-backend specific 'context information' was
to set all this data with a single assignment like this:

```c
    sg_setup(&(sg_desc){
        ...
        .context = get_context()
    });
```

This ```get_context()``` function would simply return a completely filled
out ```sg_context_desc``` structure by value.

But where should such a function live? If the API user must write this 
function in their own code, not much would be gained.

When sokol_gfx.h and sokol_app.h are used together, such a 'glue function'
needs to know the sg_context_desc struct from the sokol_gfx.h header,
and the various sokol_app.h functions to query default framebuffer
attributes. But one of the main design goals of the (core-) sokol headers
is that the headers should be standalone and not depend on each other
(only the "2nd-tier" utility headers may depend on other sokol headers).

That's where the new sokol_glue.h header comes in, this is a special header
providing helper 'glue functions' for combinations of sokol headers.

The sokol_glue.h header should always be included *after* all other sokol headers,
because it automtically detects what other sokol header have been included
and before and automatically offers matching 'extension functions'.

In a typical 'sokol_app.h + sokol_gfx.h' application it might look like this:

```c
#include "sokol_app.h"
#include "sokol_gfx.h"
#include "sokol_glue.h"
```

In the sokol_glue.h header, API functions are wrapped like this:

```c
#if defined(SOKOL_GFX_INCLUDED) && defined(SOKOL_APP_INCLUDED)
SOKOL_API_DECL sg_context_desc sapp_sgcontext(void);
#endif
```

So when both sokol_gfx.h and sokol_app.h have been included before sokol_glue.h,
a new function ```sg_context_desc sapp_sgcontext(void)``` becomes available,
which *looks* like a regular sokol_app.h function, except that it uses
a return type from the sokol_gfx.h header.

With this new glue function, the "new way" to setup sokol_gfx.h with the
3D-backend information obtained from sokol_app.h looks like this:

```c
    sg_setup(&(sg_desc){
        .context = sapp_sgcontext()
    });
```

Currently, sokol_glue.h only has this one function, but more and similar
helper functions will be added in the future for other combinations of
sokol headers (or even combinations of sokol headers with 'foreign' APIs).

And finally on to the last change:

## WebGPU shader output in sokol-shdc

The sokol-shdc shader-cross-compiler/code-generation tool now has a new
output option for WebGPU which (currently) outputs SPIRV bytecode
(later this will be changed to the new WGSL shading language):

```sh
> sokol_shdc --input shader.glsl --output shader.h --slang wgpu
```

This creates a C header with embedded SPIRV byte code and code-generated
structs and functions for integrating the shader with sokol-gfx.

The only other noteworthy feature is that the new ```sg_sampler_type``` enum
I already talked about above is extracted from the shader's texture sampler
slots using the SPIRVCross reflection API.

Existing 'annotated GLSL shader code' doesn't require any changes. This 'no
changes' requirement was actually a bit of a challenge, but WebGPU basically
"expects" that uniform-block- and texture-bindings have explicitely defined
'binding sets', in GLSL it looks like this for instance:

```
layout(set = 0, binding = 0) uniform vs_params { ... };
layout(set = 2, binding = 0) uniform sampler2D tex;
```

But such binding sets are not exposed in the sokol-gfx API, instead each
backend defines a hardwired internal mapping of the resource bind-slot model 
of sokol-gfx maps to the resource binding mechanism of a specific backend API.

The sokol-shdc shader tool also needs to be aware of those backend-specific
mapping, just in the opposite direction, it needs to know how the 3D-APIs
resource binding model maps back to the sokol-gfx binding model. To accomplish
this, it needs to add annotations (or rather in SPIRV-lingo: decorations)
to shader resources like uniform blocks and texture samplers.

This 'decoration step' happens through the SPIRVCross API after the
GLSL input source code has been compiled to SPIRV and before this SPIRV
byte code is translated into the various shading languages needed by
the sokol-gfx backends.

But there's a chicken-egg problem. Current WebGPU implementations expect
SPIRV as shader byte code, so the entire SPIRVCross translation step would
be skipped. But (from what I've seen, and I didn't look for all too long) 
only SPIRVCross has a convenient API to completely modify the resource binding
decorations (the GLSL-to-SPIRV compiler only has very broad rules to automatically
assign binding decorations, but those rules were not detailed enough for
my situation).

So in a sudden stroke of (perverted) genius I'm now compiling the
shader code **twice** from GLSL to SPIRV:

- compile the 'undecorated' GLSL source to SPIRV
- feed the result into SPIRVCross and "manually" set decoration
- have SPIRVCross translate this decorated SPIRV bytecode back to GLSL
- the resulting output GLSL source (with decoration) is again compiled to SPIRV

...not exactly elegant, but it works nicely and isn't that much slower (still
faster than - for instance - creating Metal bytecode via sokol-shdc, because
this needs to invoke the Metal compiler as external tool).

And that's all I think. Some time in the next few days I will slap a tag
on the current sokol-gfx master branch (as well as sokol-tools and sokol-samples),
and then merge the changes described here into master.

A much more detailed blog post about the new sokol-gfx WebGPU backend will
also follow soon-ish.
