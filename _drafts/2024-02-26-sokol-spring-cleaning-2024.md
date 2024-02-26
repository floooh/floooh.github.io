---
layout: post
title: "Upcoming Sokol header API changes (Feb 2024)"
---

In a couple of days I will merge the first big API update of 2024 for sokol_gfx.h (with some
collateral changes in sokol_app.h, sokol_glue.h and sokol_gfx_imgui.h).

The API update in sokol_gfx.h is **BREAKING** for all code, but for most use cases
the required changes are fairly minimal.

The overall PR is here: [#985](https://github.com/floooh/sokol/pull/985).

In addition to this blog post, please also re-read the documentation headers
in sokol_gfx.h and sokol_app.h, and specifically the struct documentation
for the new structs `sg_environment` and `sg_swapchain`.

If you use sokol_gfx.h + sokol_app.h + sokol_glue.h, check out the updated samples
here (first click on a sample, and then on the 'src' link at the bottom):

- [sokol samples](https://floooh.github.io/sokol-html5/)

Specifically look at [clear-sapp](https://floooh.github.io/sokol-html5/clear-sapp.html)
for the simple case of only rendering to a default framebuffer, and
[offscreen-sapp](https://floooh.github.io/sokol-html5/offscreen-sapp.html) for
rendering to an offscreen render target.

If you use sokol_gfx.h with your own window bindings, or a library like GLFW or SDL,
check our the backend specific examples,

- for D3D11: [https://github.com/floooh/sokol-samples/tree/master/d3d11](https://github.com/floooh/sokol-samples/tree/master/d3d11)
- for Metal: [https://github.com/floooh/sokol-samples/tree/master/metal](https://github.com/floooh/sokol-samples/tree/master/metal)
- for GL with GLFW: [https://github.com/floooh/sokol-samples/tree/master/glfw](https://github.com/floooh/sokol-samples/tree/master/glfw)
- for WASM/Emscripten: [https://github.com/floooh/sokol-samples/tree/master/html5](https://github.com/floooh/sokol-samples/tree/master/html5)

Specifically have a look at these new functions:

- D3D11: `d3d11_environment()` and `d3d11_swapchain()`
- Metal: `osx_environment()` and `osx_swapchain()`
- Emscripten: `emsc_environment()` and `emsc_swapchain()`
- GLFW: `glfw_environment()` and `glfw_swapchain()`

The GLFW subdirectory also contains an updated `multiwindow-glfw` sample, and
a `metal-glfw` sample which demonstrates how to use GLFW in NO_API mode together
with the sokol_gfx.h Metal backend.

Also please be aware of the following behaviour and expectation changes if you
are using your own window system glue:

If you're using sokol_gfx.h with your own **D3D11/DXGI** window bindings please
be aware of an important behaviour change for multisampled rendering:
Previously, the window glue code was expected to perform the MSAA resolve
operation. This has been moved into `sg_end_pass()` to be consistent with the
other backends.

On macOS and iOS, if you're using sokol_gfx.h with your own **Cocoa/Metal**
window bindings, please be aware that you are no longer expected to provide an
`MTLRenderPassDescriptor` object to describe a swapchain, but instead
individual `CAMetalDrawable` and `MTLTexture` objects. This was also done as
'harmonization' with other backends, and for window system glue code that
doesn't use the high level `MTKView`.

For **GL**, a GL framebuffer object handle is now required (this is typically
a hardwired zero for the default framebuffer).

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


## Table of Contents

* TOC
{:toc}

## Overview and Motivation

The general topic of this update is a cleanup of the sokol-gfx render pass
functions and how external swapchain information is passed into sokol-gfx.

Previously there was a 'default render pass' which was associated with a
'default framebuffer', and the concept of 'contexts' to allow rendering to
different 'default framebuffers' (very similar to traditional OpenGL contexts,
and in fact this old behaviour only ever matched OpenGL, but not the other 3D
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

Instead of being restricted to a single 'default-render-pass' per frame and context,
an application can now simply call `sg_begin_pass()` multiple times per frame,
each time with different swapchain properties, and all that without having to
create 'context objects' upfront.

Another change is that all callback functions for providing swapchain surfaces are
gone, instead swapchain surfaces are now provided as type-erased void pointers.

Here are some before-after code examples when using sokol_gfx.h, sokol_app.h and
sokol_glue.h:

Setting up sokol-gfx, before:

```c
sg_setup(&(sg_desc){
    .context = sapp_sgcontext(),
    .logger.func = slog_func,
});
```

...and after:

```c
sg_setup(&(sg_desc){
    .environment = sglue_environment(),
    .logger.func = slog_func,
});
```

The new `.environment` nested struct is essentially the old `.context` nested struct minus
the 'default swapchain properties'.

Rendering to the 'default framebuffer, before:

```c
sg_begin_default_pass(&pass_action, sapp_width(), sapp_height());
```

...and after:

```c
sg_begin_pass(&(sg_pass){
    .action = pass_action,
    .swapchain = sglue_swapchain(),
});
```

For simple applications which don't render to offscreen passes, these two changes
are all that's needed to make old code work.

For other cases, read on.

## Detailed change list

### sokol_gfx.h

The following public API structs and functions have been **removed**:

- `sg_begin_default_pass()`
- `sg_begin_default_passf()`
- `struct sg_context`
- `sg_setup_context()`
- `sg_activate_context()`
- `sg_discard_context()`

The following top-level struct are **new**:

- `struct sg_environment`: this is passed as nested struct of `sg_desc` into
  the `sg_setup()` call to provide information about the environment sokol-gfx
  runs in (most importantly 3D API device pointers).

- `struct sg_swapchain`: this is now passed into `sg_begin_pass()` for render passes
  which should render into an externally managed swapchain. The struct contains the
  following information:
    - the pixel format of the color rendering surface
    - the pixel format of the optional depth/stencil rendering surface
    - an MSAA sample count
    - 3D backend specific resource handles, like texture views, 'drawables'
      and framebuffers

The resource type `sg_pass` has been **renamed** to `sg_attachments` (to free the name
for another purpose), this in turn causes the following related renames:

- `sg_pass` => `sg_attachments`
- `sg_pass_desc` => `sg_attachments_desc`
- `sg_pass_info` => `sg_attachments_info`
- `sg_make_pass()` => `sg_make_attachments()`
- `sg_destroy_pass()` => `sg_destroy_attachmnts()`
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

With the struct `sg_pass` now looking like this (without the start/end canaries):

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

...and for a 'swapchain-render-pass', a '.swapchain` item must be provided, but no
`.attachments`:

```c
sg_begin_pass(&(sg_pass){
    .action = pass_action,
    .swapchain = sglue_swapchain(),
});
```

If you don't use sokol_gfx.h together with sokol_app.h and sokol_glue.h, it is recommended
that you create a similar helper function like `sglue_swapchain()` which returns
a filled-out `sg_swapchain` struct by value which can be plugged directly into
`sg_pass.swapchain`.

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

The sokol_glue.h is now a bit 'less weird' in that it doesn't provide a tailored public
API depending on what other headers have been included before it (it was an interesting
idea that was essentially pointless because sokol_glue.h never became more than glue
code between sokol_app.h and sokol_gfx.h).

The API prefix has also changed from a somewhat confusing `sapp_` to the expected `sglue_`.

The only old API function `sapp_sgcontext()` has been split into two new functions:

- `sglue_environment()` which plugs directly into `sg_desc.environment`, and...
- `sglue_swapchain()` which plugs into `sg_pass.swapchain`

`sglue_swapchain()` will return different values each frame, so don't accidentally
cache the result anywhere.

### sokol_gfx_imgui.h

In a similar vein, the public API prefix of sokol_gfx_imgui.h has been changed from the
weird 'double prefix' `sg_imgui_` to the expected `sgimgui_`.

Apart from the this publicly visible change, all the internals have been updated to reflect
the sokol-gfx API changes.

## Q: Why still have a baked pass attachments object?

I've been pondering for a little bit to get rid of pre-baked pass-attachments
objects alltogether (e.g. what were formerly `sg_pass` objects and are now
`sg_attachments` objects), and instead would essentially pass a transient
struct with the same information that's in `sg_attachments_desc` into the
`sg_begin_pass()` function, similar to how `sg_apply_bindings()` takes
a transient `sg_bindings` struct will all the resource bindings.

I didn't follow through with that idea because this would mean creating
temporary objects inside `sg_begin_pass()` and discarding them again in
`sg_end_pass()` (or alternatively use a 'hash-and-cache' approach).

In D3D11 and WebGPU, one temporary texture view object would need
to be created per pass-attachment (which may add up to 9 objects),
and in the GL backend, a GL framebuffer object must be created,
configured and checked for completeness. All this work currently
only once in `sg_make_attachments()`, but would need to happen
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

## Change Recipes for sokol_gfx.h + sokol_app.h

[TODO]

## Change Recipes for sokol_gfx.h + custom window system glue

[TODO]
