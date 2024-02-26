---
layout: post
title: "Upcoming Sokol header API changes (Feb 2024)"
---

In a couple of days I will merge the first big API update of 2024 for sokol_gfx.h (with some
related changes in sokol_app.h, sokol_glue.h and sokol_gfx_imgui.h).

> NOTE: most links to code examples will only point to the right code after [PR #985](https://github.com/floooh/sokol/pull/985) has been merged!

The API update in sokol_gfx.h is a **BREAKING CHANGE** for all code, but for most use cases
the required changes are fairly minimal.

Apologies for the broken syntax highlighting, apparently [Rouge](https://github.com/rouge-ruby/rouge) doesn't understand C99.

## Table of Contents

* TOC
{:toc}

## Overview and Motivation

The general topic of this update is a cleanup of the sokol-gfx render pass
functions and how external swapchain information is passed into sokol-gfx.

Previously there was a special 'default render pass' into a 'default framebuffer',
and the concept of 'contexts' to allow switching between different rendering contexts
and their default framebuffers (very similar to traditional OpenGL contexts,
and in fact this old behavior only ever matched OpenGL, but not the other backend
APIs).

This setup was needlessly complicated for people who want to use sokol-gfx
to render into multiple windows, leading to planning [ticket #904](https://github.com/floooh/sokol/issues/904),
and then to [PR #985](https://github.com/floooh/sokol/pull/985).

The gist is:

- There is now only a single 'unified' `sg_begin_pass()` function which covers
  both rendering into sokol-gfx render target textures (aka 'offscreen passes')
  and externally managed 'swapchains' (aka 'swapchain passes').
- The entire concept of `contexts` has been removed from sokol_gfx.h.
- External swapchain properties are now passed directly into `sg_begin_pass()`
  in a transient structure.

Instead of having a special and unique 'default-render-pass' per frame and context,
an application can now simply call `sg_begin_pass()` multiple times per frame,
each time with properties for a different swapchain, and all that without having to
create 'context objects' upfront or 'switching contexts'.

Most simple applications that don't render into offscreen passes and
use sokol_gfx.h together with sokol_app.h and sokol_glue.h only need to change
two calls: `sg_setup()` and `sg_begin_default_pass()`, for other situations
please check the 'Change Recipes' section further down.

In addition to this blog post, please also re-read the documentation headers
in sokol_gfx.h and sokol_app.h, and specifically the struct documentation
for the new sokol-gfx structs `sg_environment` and `sg_swapchain`.

## Detailed change list

### sokol_gfx.h

The following public API structs and functions have been **removed**:

- `sg_begin_default_pass()`
- `sg_begin_default_passf()`
- `struct sg_context`
- `sg_setup_context()`
- `sg_activate_context()`
- `sg_discard_context()`

The following top-level structs have been **added**:

- `struct sg_environment`: this is passed as a nested struct of `sg_desc` into
  the `sg_setup()` call to provide information about the environment sokol-gfx
  runs in (most importantly 3D API device pointers).

- `struct sg_swapchain`: this is passed into `sg_begin_pass()` for render passes
  which should render into an externally managed swapchain. The struct contains the
  following information:
    - the pixel format of the swapchain's rendering surface
    - the pixel format of the optional depth/stencil surface
    - an MSAA sample count
    - 3D backend specific resource handles, like D3D11/WebGPU texture views, Metal drawables,
      or GL framebuffers

The resource handle type `sg_pass` has been **renamed** to `sg_attachments` (to
free the name for another purpose), this also causes related renames:

- `sg_pass` => `sg_attachments`
- `sg_pass_desc` => `sg_attachments_desc`
- `sg_pass_info` => `sg_attachments_info`
- `sg_make_pass()` => `sg_make_attachments()`
- `sg_destroy_pass()` => `sg_destroy_attachments()`
- `sg_query_pass_state()` => `sg_query_attachments_state()`
- `sg_query_pass_info()` => `sg_query_attachments_info()`
- `sg_query_pass_desc()` => `sg_query_attachments_desc()`
- `sg_alloc_pass()` => `sg_alloc_attachments()`
- `sg_dealloc_pass()` => `sg_dealloc_attachments()`
- `sg_init_pass()` => `sg_init_attachments()`
- `sg_fail_pass()` => `sg_fail_attachments()`
- `sg_[*]_pass_info()` => `sg_[*]_attachments_info()` (where '*' is 'd3d11|gl|metal|wgpu')

Inside the `sg_attachments_desc` struct there has been some renaming to reduce redundancy:

- `.color_attachments[]` => `.colors[]`
- `.resolve_attachments[]` => `.resolves[]`
- `.depth_stencil_attachment` => `.depth_stencil`

The typename `sg_pass` has been repurposed to serve as the `sg_begin_pass()` argument,
e.g. the begin-pass function signature now looks like this:

```c
void sg_begin_pass(const sg_pass* pass);
```

With the struct `sg_pass` now looking like this (with omitted start/end canaries):

```c
typedef struct sg_pass {
    sg_pass_action action;
    sg_attachments attachments;
    sg_swapchain swapchain;
    const char* label;
} sg_pass;
```

For an 'offscreen-render-pass', an `.attachments` item must be provided, but no
`.swapchain`:

```c
sg_begin_pass(&(sg_pass){
    .action = pass_action,
    .attachments = attachments,
});
```

...and for a 'swapchain-render-pass', a `.swapchain` item must be provided, but no
`.attachments`:

```c
sg_begin_pass(&(sg_pass){
    .action = pass_action,
    .swapchain = sglue_swapchain(),
});
```

Other unrelated 'drive-by-changes' in sokol_gfx.h:

- `sg_limits.gl_max_vertex_uniform_vectors` has been replaced with `sg_limits.gl_max_vertex_uniform_components`
  (see [#714](https://github.com/floooh/sokol/issues/714).
- the start and end canaries in `sg_pass_action` have been removed (since `sg_pass_action` is now a nested
  struct of `sg_pass`, the canaries are redundant)
- a new initialization config item `sg_desc.mtl_use_command_buffer_with_retained_references` has been added,
  (see: [#981](https://github.com/floooh/sokol/issues/981))

### sokol_app.h

The following public API function has been removed:

- `sapp_metal_get_renderpass_descriptor()`

The following functions have been renamed:

- `sapp_metal_get_drawable()` => `sapp_metal_get_current_drawable()`
- `sapp_d3d11_get_render_target_view()` => `sapp_d3d11_get_render_view()`

...and the following functions are new:

- `sapp_metal_get_depth_stencil_texture()`
- `sapp_metal_get_msaa_color_texture()`
- `sapp_d3d11_get_resolve_view()`
- `sapp_gl_get_framebuffer()`

...These functions directly plug into the new `sg_swapchain` struct in sokol_gfx.h.

### sokol_glue.h

sokol_glue.h is now a regular library header without the 'preprocessor magic'
which created a different API depending on what other sokol headers had been
included before sokol_glue.h (this was an 'interesting' but in ultimately
pretty stupid idea).

The API prefix has changed from a somewhat confusing `sapp_` to the expected `sglue_`.

The old function `sapp_sgcontext()` has been split into two new functions:

- `sglue_environment()` which plugs directly into `sg_desc.environment`, and...
- `sglue_swapchain()` which plugs into `sg_pass.swapchain`

Note that `sglue_swapchain()` will return different values each frame.

### sokol_gfx_imgui.h

In a similar vein, the public API prefix of sokol_gfx_imgui.h has been changed from the
weird 'double prefix' `sg_imgui_` to a more conventional `sgimgui_`.

Apart from this publicly visible change, all the internals have been updated to reflect
the sokol-gfx API changes.

## Link collection with example code changes

If you use sokol_gfx.h + sokol_app.h + sokol_glue.h, check out the updated samples
here (first click on a sample, and then on the 'src' link at the bottom):

- [sokol samples](https://floooh.github.io/sokol-html5/)

Specifically look at [clear-sapp](https://floooh.github.io/sokol-html5/clear-sapp.html)
for the simple case of only rendering to a default framebuffer, and
[offscreen-sapp](https://floooh.github.io/sokol-html5/offscreen-sapp.html) for
rendering to an offscreen render target.

If you use sokol_gfx.h with your own window system glue, or a library like GLFW or SDL,
check out the updated backend specific examples:

- for D3D11: [https://github.com/floooh/sokol-samples/tree/master/d3d11](https://github.com/floooh/sokol-samples/tree/master/d3d11)
- for Metal: [https://github.com/floooh/sokol-samples/tree/master/metal](https://github.com/floooh/sokol-samples/tree/master/metal)
- for GL with GLFW: [https://github.com/floooh/sokol-samples/tree/master/glfw](https://github.com/floooh/sokol-samples/tree/master/glfw)
- for WASM/Emscripten: [https://github.com/floooh/sokol-samples/tree/master/html5](https://github.com/floooh/sokol-samples/tree/master/html5)

The GLFW subdirectory also contains an updated `multiwindow-glfw` sample, and
a `metal-glfw` sample which demonstrates how to use GLFW in NO_API mode together
with the sokol_gfx.h Metal backend.

Also please be aware of the following behaviour and expectation changes if you
are using your own window system glue:

- For **D3D11/DXGI** please be aware of an important behavior change for
  multisampled rendering: Previously, the window glue code was expected to
  perform the MSAA resolve operation. This has been moved into `sg_end_pass()`
  to be consistent with the other backends.

- For **Metal** please be aware that you are no longer expected to provide
  an `MTLRenderPassDescriptor` object to describe a swapchain, but instead
  individual `CAMetalDrawable` and `MTLTexture` objects. This was also done as
  'harmonization' with other backends.

- For **GL**, sokol-gfx now expects that *all* rendering goes through a single
  GL context. This may require changes to existing code which renders into
  multiple windows (for instance in GLFW, every window has its own GL context).
  Refer to the new
  [multiwindow-glfw.c](https://github.com/floooh/sokol-samples/blob/master/glfw/multiwindow-glfw.c)
  example for a possible solution.

Additionally, check out the following PRs for required changes in my toy
projects:

- [pacman.c](https://github.com/floooh/pacman.c/pull/12)
- [Doom on Sokol](https://github.com/floooh/doom-sokol/pull/1)
- [Visual 6502 Remix](https://github.com/floooh/v6502r/pull/24/files)
- [qoiview](https://github.com/floooh/qoiview/pull/10)
- [chips](https://github.com/floooh/chips-test/pull/33)

When using the language bindings, check out the following PRs:

- [sokol-zig](https://github.com/floooh/sokol-zig/pull/57/files)
- [sokol-odin](https://github.com/floooh/sokol-odin/pull/8)
- [sokol-nim](https://github.com/floooh/sokol-nim/pull/28)
- [sokol-rust](https://github.com/floooh/sokol-rust/pull/22)
- [pacman.zig](https://github.com/floooh/pacman.zig/pull/23)
- [kc85.zig](https://github.com/floooh/kc85.zig/pull/4)


## Detailed Change Recipes

### ...for sokol_gfx.h + sokol_app.h + sokol_glue.h

When using sokol_gfx.h together with sokol_app.h and sokol_glue.h...

...change your `sg_setup()` call from this:

```c
sg_setup(&(sg_desc){
    .context = sapp_sgcontext(),
    .logger.func = slog_func,
});
```

...to this:

```c
sg_setup(&(sg_desc){
    .environment = sglue_environment(),
    .logger.func = slog_func,
});
```

Change the `sg_begin_default_pass()` call from this:

```c
sg_begin_default_pass(&pass_action, sapp_width(), sapp_height());
```

...to this:

```c
sg_begin_pass(&(sg_pass){
    .action = pass_action,
    .swapchain = sglue_swapchain()
});
```

### ...for offscreen render passes

Change `sg_make_pass()` calls from this:

```c
sg_pass pass = sg_make_pass(&(sg_pass_desc){
    .color_attachments[0].image = color_img,
    .resolve_attachments[0].image = resolve_img,
    .depth_stencil_attachment.image = depth_img,
});
```

...to this:

```c
sg_attachments attachments = sg_make_attachments(&(sg_attachments_desc){
    .colors[0].image = color_img,
    .resolves[0].image = resolve_img,
    .depth_stencil.image = depth_img,
});
```

Change `sg_begin_pass()` calls from this:

```c
sg_begin_pass(pass, &pass_action);
```

...to this:

```c
sg_begin_pass(&(sg_pass){
    .action = pass_action,
    .attachments = attachments,
});
```

### ...for custom window system glue

Create two helper functions, one which returns an initialized `sg_environment`
struct and one which returns an initialized `sg_swapchain` struct. Following
are examples how these functions might look like for different backend 3D APIs.

#### ...using D3D11:

Example implementations:

```c
sg_environment d3d11_environment(void) {
    return (sg_environment){
        .defaults = {
            .color_format = SG_PIXELFORMAT_BGRA8,
            .depth_format = SG_PIXELFORMAT_DEPTH_STENCIL,
            .sample_count = 4,
        },
        .d3d11 = {
            .device = d3d11_device, // ID3D11Device*
            .device_context = d3d11_device_context, // ID3D11DeviceContext*
        }
    };
}
```

`.defaults.color_format`, `defaults.depth_format` and `defaults.sample_count` must match the swapchain
surface properties. These defaults will be used to fill in defaults for zero-initialized
values in various sokol-gfx calls. `.depth_format` can also be `SG_PIXELFORMAT_NONE` if
no depth-buffer exists, or `SG_PIXELFORMAT_DEPTH` if no stencil buffer is used.

The associated DXGI depth-stencil-view pixel formats are:

- `SG_PIXELFORMAT_DEPTH_STENCIL` => `DXGI_FORMAT_D24_UNORM_S8_UINT`
- `SG_PIXELFORMAT_DEPTH` => `DXGI_FORMAT_D32_FLOAT`

The helper function to obtain an `sg_swapchain` struct might look like this:

```c
sg_swapchain d3d11_swapchain(void) {
    return (sg_swapchain){
        .width = state.width,
        .height = state.height,
        .sample_count = state.sample_count,
        .color_format = SG_PIXELFORMAT_BGRA8,
        .depth_format = SG_PIXELFORMAT_DEPTH_STENCIL,
        .d3d11 = {
            .render_view = (state.sample_count == 1) ? state.rt_view : state.msaa_view,
            .resolve_view = (state.sample_count == 1) ? 0 : state.rt_view,
            .depth_stencil_view = state.ds_view,
        }
    };
}
```

`state.rt_view` and `state.msaa_view` are of type `ID3D11RenderTargetView` and `state.ds_view` is
of type `ID3D11DepthStencilView`.

Note how a different `.d3d11.render_view` is selected depending on whether multisampled rendering
is used or not. For non-multisampled rendering, sokol-gfx renders into the same view that's
presented. For multisampled rendering, sokol-gfx will render into an intermediate MSAA texture
view (`state.msaa_view`) which is then resolved into the presented `.render_view` inside
`sg_end_pass()` (the MSAA resolve happening inside `sg_end_pass()` instead of the window system
glue is actually different behavior from before).

Also check out the example D3D11 window system glue code here:

[https://github.com/floooh/sokol-samples/blob/master/d3d11/d3d11entry.c](https://github.com/floooh/sokol-samples/blob/master/d3d11/d3d11entry.c)

#### ...using Metal

Example function which returns an initialized `sg_environment` struct:

```c
sg_environment osx_environment(void) {
    return (sg_environment) {
        .defaults = {
            .sample_count = sample_count,
            .color_format = SG_PIXELFORMAT_BGRA8,
            .depth_format = SG_PIXELFORMAT_DEPTH_STENCIL,
        },
        .metal = {
            .device = (__bridge const void*) mtl_device,
        }
    };
}
```

The ObjC type of `mtl_device` is `id<MTLDevice>`. Note the special `__bridge` cast to
a void pointer for tunneling through the sokol_app.h and sokol_gfx.h C APIs.

...and the function which returns an `sg_swapchain` struct (in this case using an `MTKView`
to manage the swapchain surfaces):

```c
sg_swapchain osx_swapchain(void) {
    return (sg_swapchain) {
        .width = (int) [mtk_view drawableSize].width,
        .height = (int) [mtk_view drawableSize].height,
        .sample_count = sample_count,
        .color_format = SG_PIXELFORMAT_BGRA8,
        .depth_format = SG_PIXELFORMAT_DEPTH_STENCIL,
        .metal = {
            .current_drawable = (__bridge const void*) [mtk_view currentDrawable],
            .depth_stencil_texture = (__bridge const void*) [mtk_view depthStencilTexture],
            .msaa_color_texture = (__bridge const void*) [mtk_view multisampleColorTexture],
        }
    };
}
```

Also check out the Metal window system glue code here:

[https://github.com/floooh/sokol-samples/blob/master/metal/osxentry.m](https://github.com/floooh/sokol-samples/blob/master/metal/osxentry.m)

...alternatively check out the GLFW+Metal example here which doesn't use an MTKView (but also doesn't
support a depth-buffer or MSAA rendering):

[https://github.com/floooh/sokol-samples/blob/master/glfw/metal-glfw.m](https://github.com/floooh/sokol-samples/blob/master/glfw/metal-glfw.m)

#### ...using WebGPU

The environment- and swapchain-helper-functions look very similar to D3D11:

```c
sg_environment wgpu_environment(void) {
    return (sg_environment) {
        .defaults = {
            .color_format = SG_PIXELFORMAT_...,
            .depth_format = SG_PIXELFORMAT_...,
            .sample_count = state.desc.sample_count,
        },
        .wgpu = {
            .device = (const void*) state.device,
        }
    };
}

```

For `.defaults.color_format` you should use the result of
`wgpuSurfaceGetPreferredFormat()` translated to a sokol-gfx pixel format
(either `SG_PIXELFORMAT_BGRA8` or `SG_PIXELFORMAT_RGBA8`).

For the depth format use either `SG_PIXELFORMAT_DEPTH_STENCIL`,
`SG_PIXELFORMAT_DEPTH` or `SG_PIXELFORMAT_NONE`, which translate to WebGPU
pixel formats as follows:

- `SG_PIXELFORMAT_DEPTH_STENCIL` => `WGPUTextureFormat_Depth32FloatStencil8`
- `SG_PIXELFORMAT_DEPTH` => `WGPUTextureFormat_Depth32Float`

The type of `state.device` is `WGPUDevice`.

The WebGPU swapchain helper function might look like this:

```c
sg_swapchain wgpu_swapchain(void) {
    return (sg_swapchain) {
        .width = state.width,
        .height = state.height,
        .sample_count = state.sample_count,
        .color_format = SG_PIXELFORMAT_...,
        .depth_format = SG_PIXELFORMAT_...,
        .wgpu = {
            .render_view = (state.sample_count == 1) state.rt_view : state.msaa_view,
            .resolve_view = (state.sample_count == 1) ? 0 : state.rt_view,
            .depth_stencil_view = state.ds_view,
        }
    };
}
```

...note the selection for `.wgpu.render_view` and `.wgpu.resolve_view` based on the MSAA
sample count.

The types for all view objects are `WGPUTextureView`.

Also check out the WebGPU system glue code here:

[https://github.com/floooh/sokol-samples/blob/master/wgpu/wgpu_entry.c](https://github.com/floooh/sokol-samples/blob/master/wgpu/wgpu_entry.c)

#### ...GL with GLFW

The environment-helper-function only returns default pixel formats and sample count:

```c
sg_environment glfw_environment(void) {
    return (sg_environment) {
        .defaults = {
            .color_format = SG_PIXELFORMAT_RGBA8,
            .depth_format = SG_PIXELFORMAT_DEPTH_STENCIL,
            .sample_count = 4,
        },
    };
}
```

...the swapchain function also returns a GL framebuffer object, for the default framebuffer
this is always zero, otherwise this is a handle created with `glGenFramebuffers()`.

```c
sg_swapchain glfw_swapchain(void) {
    int width, height;
    glfwGetFramebufferSize(_window, &width, &height);
    return (sg_swapchain) {
        .width = width,
        .height = height,
        .sample_count = _sample_count,
        .color_format = SG_PIXELFORMAT_RGBA8,
        .depth_format = SG_PIXELFORMAT_DEPTH_STENCIL,
        .gl = {
            .framebuffer = 0,
        }
    };
}
```

Also see [https://github.com/floooh/sokol-samples/blob/master/glfw/glfw_glue.c](https://github.com/floooh/sokol-samples/blob/master/glfw/glfw_glue.c)

## Q: Why still have a baked pass attachments object?

I've been pondering for a little bit to get rid of pre-baked pass-attachments
objects alltogether (e.g. what were formerly `sg_pass` objects and are now
`sg_attachments` objects), and instead pass a transient struct with the same
information that's in `sg_attachments_desc` into the `sg_begin_pass()`
function, similar to how `sg_apply_bindings()` takes a transient `sg_bindings`
struct will all the resource bindings.

I didn't follow through with that idea because this would mean creating
temporary objects inside `sg_begin_pass()` and discarding them again in
`sg_end_pass()` (or alternatively use a 'hash-and-cache' approach).

In D3D11 and WebGPU, one temporary texture view object would need
to be created per pass-attachment (which may add up to 9 temporary objects),
and in the GL backend, a GL framebuffer object must be created,
configured and checked for completeness. All this work currently
only happens once in `sg_make_attachments()`, but would need to happen
inside `sg_begin_pass()` without baked attachments objects.

While these backend API objects should be 'reasonably cheap' to create, I still
decided against it.

Currently the only other place where such temporary objects are created and
discarded on the fly are in the `sg_apply_bindings()` call for the WebGPU
backend, where temporary BindGroup objects are created and discarded
dynamically via a 'hash-and-cache' approach and I hate it :) I don't want that
type of code to creep into other places.

Now, `sg_begin_pass()` and `sg_end_pass()` are by far not as high-frequency-calls as
`sg_apply_bindings()`, and creating view- and framebuffer-objects *should* be
cheap enough, but it still feels 'wrong' to create and discard backend API
objects willy-nilly during the frame.
