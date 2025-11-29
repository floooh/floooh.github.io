---
layout: post
title: "Experimental Sokol Vulkan backend"
---

In a couple of days I will merge the first iteration of a sokol-gfx
Vulkan backend. Please consider this backend as 'experimental', it has
only received limited testing, has limited platform coverage and some
known known shortcomings and feature gaps which I will address in followup
updates.

The currently known limitiations are:

- the entire code essentially expects a 'modern desktop GPU feature set',
  I haven't paid much attention to and in some cases outright ignored limitations
  of mobile GPUs
- the window system glue in sokol_app.h is only implemented for Linux/X11
- only tested on an Intel Meteor Lake integrated GPU (which also means
  that some buffer types may be allocated in memory types that are not
  optimal on GPUs without unified memory)
- barriers for CPU => GPU updates are currently quite pessimistic
  (e.g. more barriers might be inserted than needed, or at a too
  early point in a frame)
- there's currently no GPU memory allocator, nor a way to hook in
  an external GPU memory allocator like VMA (at least the latter is planned)
- rendering is currently only supported to a single swapchain
  (not a problem when used with sokol_app.h because that also only
  supports a single window)
- it's currently not possible to inject native Vulkan buffers and images
  into sokol-gfx (that's a somewhat esoteric feature supported by
  the other backends)
- I couldn't get RenderDoc to work, but it's unclear why (more details
  on that later)

On the upside:

- no sokol-gfx API or shader-authoring changes are required
  (there are some minor breaking API changes because of some code cleanup work
  I had planned already and which are not directly related to Vulkan, but
  most code should work without changes)
- the Vulkan validation layer is silent on all sokol-samples (which try to cover
  most sokol-gfx features and their combined use), and this includes the tricky
  optional synchronization2 validations (I'm pretty proud of that considering
  that most Vulkan samples I found have sync-validation errors)
- performance on my Intel Meteor Lake laptop in the [drawcallperf-sample](https://floooh.github.io/sokol-html5/drawcallperf-sapp.html)
  is already slightly better than the OpenGL backend (on a vanilla
  Kubuntu system)

It's also important to understand what actually motivated the Vulkan backend
(e.g. why now, and not earlier or much later):

It's *not* mainly about performance, but about 'future potential' and OpenGL
rot. Essentially, the Vulkan backend is the first step towards deprecating the
OpenGL backend (first, an alternative to WebGL2 had to happen - which
exists now with WebGPU, and next an alternative for OpenGL on Linux (and less
important: Android) had to be found, because Linux and Android are the only two
target platforms which are currently limited to a single 3D API: OpenGL.
All other target platforms already have a more modern alternative
(Windows with D3D11 and macOS/iOS with Metal). Deprecating the OpenGL backend
won't happen for a while, but personally I can't wait to free sokol-gfx from the
'shackles of OpenGL' ;)

Also another reason why I felt that now is the right time to tackle Vulkan support
is that the Vulkan API has improved quite a bit since 1.0 in ways that make it a much
better fit for sokol-gfx, in a nutshell (if you already know Vulkan concepts),
the sokol-gfx backend makes use of the following 'modern' features:

- 'dynamic rendering' (e.g. render passes are demarcated by begin/end
  calls instead of being baked into render-pass objects) - e.g. pretty much
  a copy of the Metal render pass model. This is a perfect match for
  sokol-gfx sg_begin_pass()/sg_end_pass()
- `EXT_descriptor_buffer` - this is a controversial choice, but it's a perfect
  match for the sokol-gfx resource binding model and I really did not want to deal
  with the traditional rigid Vulkan descriptor API (which is an overengineered boondoggle
  if I've ever seen one). This is also the main reason on why Android had to be left out
  for now, and apparently descriptor buffers are also a poor match for NVIDIA
  GPUs. The plan here is to wait until Khronos completes work on a
  descriptor pool replacement which AFAIK will be a mix of descriptor buffers and
  D3D12-style descriptor heaps and then port the `EXT_descriptor_buffer` code
  over to that new resource binding API.
- 'synchronization2' (not a drastic change from the original barrier model,
  I'm just listing it here for completeness)

Work on the Vulkan backend spans three sub-projects:

- `sokol-shdc`: added a Vulkan-flavoured SPIRV target
- `sokol_app.h`: device creation, swapchain and frame loop
- `sokol_gfx.h`: rendering and compute features

## sokol-shdc changes

From the outside, the shader compiler changes are minimal (so minimal that
the update is actually already live for a little while).

The only change is that a new output shader format has been added: `spirv_vk`
for 'Vulkan-flavoured SPIRV.  To compile a GLSL input shader to SPIRV:

```
sokol-shdc -i bla.glsl -o bla.h -l spirv_vk
```

Internally the changes are also fairly small since sokol-shdc input shaders
are already authored in 'Vulkan-flavoured GLSL', the only missing information
is the descriptor set for resource bindings.

Sokol-shdc shaders only declare a bindslot on resource bindings with
different 'bind spaces' for uniform blocks and anything else, for instance:

```glsl
layout(binding=0) uniform fs_params { ... };
layout(binding=0) uniform texture2D tex;
layout(binding=1) uniform sampler smp;
```

Sokol-shdc performs a backend-specific bindslot allocation which for SPIRV output
simply assigns descriptor sets (uniform blocks live in descriptor set 0 and
everything else in descriptor set 1), so the above code snippet essentially
becomes:

```glsl
layout(set=0, binding=0) uniform fs_params { ... };
layout(set=1, binding=0) uniform texture2D tex;
layout(set=1, binding=1) uniform sampler smp;
```

The one thing that's not straightforward is that sokol-shdc does a 'double-tap'
for SPIRV-output:

- the input shader code is compiled from GLSL to SPIRV
- SPIRVTools optimizer passes are applied to the SPIRV
- bindings are remapped (in this case: simply add descriptor set decorators
  but keep the bindslots intact)
- the SPIRV is translated back to GLSL via SPIRVCross
- finally the SPIRVCross output is compiled *again* to SPIRV

The weird double compilation is a compromise to keep the basic
sokol-shdc pipeline and make the Vulkan shader pipeline less of a special
case. Essentially, SPIRV is first used as an intermediate format in the
first compile pass, and then as output bytecode format in the second
pass.

## sokol_app.h changes

Apart from the actual Vulkan-related update I took the opportunity to do some
public API cleanup which was rolling around in my head for a while.

First, the backend-specific config options in the `sapp_desc` struct are
now grouped into per-backend-nested structs, e.g. from this:

```c
sapp_desc sokol_main(int argc, char* argv[]) {
    return (sapp_desc){
        // ...
        .win32_console_utf8 = true,
        .win32_console_attach = true,
        .html5_bubble_mouse_events = true,
        .html5_use_emsc_set_main_loop = true,
    };
}
```

...to this:

```c
sapp_desc sokol_main(int argc, char* argv[]) {
    return (sapp_desc){
        // ...
        .win32 = {
          .console_utf8 = true,
          .console_attach = true,
        },
        .html5 = {
          .bubble_mouse_events = true,
          .use_emsc_set_main_loop = true,
        }
    };
}
```

A new enum `sapp_pixel_format` has been introduced which will play a bigger
role in the future to allow more configuration options for the sokol-app swapchain.

A ton of backend-specific functions to query backend-specific objects have been
merged to better harmonize with sokol-gfx:

The following functions to obtain backend-specific 3D API objects have been removed:

```c
const void* sapp_metal_get_device(void);
const void* sapp_metal_get_current_drawable(void);
const void* sapp_metal_get_depth_stencil_texture(void);
const void* sapp_metal_get_msaa_color_texture(void);
const void* sapp_d3d11_get_device(void);
const void* sapp_d3d11_get_device_context(void);
const void* sapp_d3d11_get_render_view(void);
const void* sapp_d3d11_get_resolve_view(void);
const void* sapp_d3d11_get_depth_stencil_view(void);
const void* sapp_wgpu_get_device(void);
const void* sapp_wgpu_get_render_view(void);
const void* sapp_wgpu_get_resolve_view(void);
const void* sapp_wgpu_get_depth_stencil_view(void);
uint32_t sapp_gl_get_framebuffer(void);
```

...and replaced with two new functions:

```c
sapp_environment sapp_get_environment(void);
sapp_swapchain sapp_get_swapchain(void);
```

The new structs `sapp_environment` and `sapp_swapchain` conceptually plug into
the sokol-gfx structs `sg_environment` and `sg_swapchain` (with the emphasis
on **conceptually**, you still need a mapping from the sokol-app structs to the
sokol-gfx structs, and this mapping is still peformed by the sokol_glue.h header.

That's it for the public API changes in sokol_app.h, now on to the Vulkan
specific parts:

The new struct `sapp_environment` contains a nested struct
`sapp_vulkan_envirnonment vulkan;` with Vulkan object pointers (as type-erased
void-pointers so that they can be tunneled through backend-agnostic code):

```c
typedef struct sapp_vulkan_environment {
    const void* physical_device;  // vkPhysicalDevice*
    const void* device;           // vkDevice*
    const void* queue;            // vkQueue*
    uint32_t queue_family_index;
} sapp_vulkan_environment;
```

...and likewise the new struct `sapp_swapchain` contains a nested struct
`sapp_vulkan_swapchain vulkan;` with Vulkan object pointers which are relevant
for a sokol-gfx swapchain render pass:

```c
typedef struct sapp_vulkan_swapchain {
    const void* render_image;           // vkImage
    const void* render_view;            // vkImageView
    const void* resolve_image;          // vkImage;
    const void* resolve_view;           // vkImageView
    const void* depth_stencil_image;    // vkImage
    const void* depth_stencil_view;     // vkImageView
    const void* render_finished_semaphore;  // vkSemaphore
    const void* present_complete_semaphore; // vkSemaphore
} sapp_vulkan_swapchain;
```

The Vulkan-specific startup code path looks like this (the usual boilerplate-heavy
initialization dance):

- A `VkInstance` object is created.
- A platform- and window-system-specific `vkSurfaceKHR` object is created,
  this is essentially the glue between a Vulkan swapchain and a specific
  window system. In the first release this window system glue code is only
  implemented for X11 via `vkCreateXlibSurfaceKHR`.
- A `vkPhysicalDevice` is picked, this is the first time where the sokol-app
  backend takes a couple of shortcuts, initialization will fail if:
  - EXT_descriptor_buffer is not supported (this currently rules out
    most mobile devices)
  - the supported Vulkan API version is not at least 1.3
  - no 'queue family' exists which supports graphics, compute, transfer
    and presentation operations all on the same queue
- Next a logical `vkDevice` object is created with the following required
  features and extensions (with the exception of compressed texture formats
  which are optional):
  - a single queue
  - EXT_descriptor_buffer
  - extendedDynamicState
  - bufferDeviceAddress
  - dynamicRendering
  - synchronization2
  - samplerAnisotropy
  - optional:
    - textureCompressionBC
    - textureCompressionETC2
    - textureCompressionASTC_LDR
- The swapchain is initialized:
  - a `VkSwapchainKHR` object is created:
    - pixel format currently either RGBA8 or BGRA8 (no sRGB)
    - present mode hardwired to `VK_PRESENT_MODE_FIFO_KHR`
    - composite-alpha hardwired to `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`
    - `VkImage` and `VkImageView` objects are obtained or created
      for the swapchain images, depth-stencil-attachment
      and optional MSAA rendering surface
- Finally a couple of VkSemaphore objects are created for each
  stage in the 'swapchain-pipeline':
    - one `render_finished_semaphore` which signals that the GPU
      has finished rendering to a swapchain surface
    - one `present_complete_semaphore` which signals that presenting
      a swapchain surface has completed and is ready for reuse

At this point, the Vulkan specific code in sokol_app.h is at about 600
lines of code, which is a lot of boilerplate, but OTH is a lot less messy
than the combined OpenGL window system code for GLX, EGL, WGL or NSOpenGL.

The actually interesting stuff happens in the last two Vulkan backend functions:

The internal function `_sapp_vk_swapchain_next()` is a wrapper around
`vkAcquireNextImageKHR()` and obtains the next free swapchain image. The
function will also signal the `present_complete_semaphore`.

The last function in the sokol-app Vulkan backend is `_sapp_vk_present()`,
this is a wrapper for `vkQueuePresentKHR()`. The present operation waits
for the `render_finished_sem` of the current frame (basically waiting until
the GPU has finished rendering the current frame). When the `vkQueuePresentKHR()`
function returns with `VK_ERROR_OUT_OF_DATE_KHR` or `VK_SUBOPTIMAL_KHR`, the
swapchain resources are recreated (this happens for instance when the window
is resized).

There's a couple of open todo points in the sokol-app Vulkan backend which
I'll take care of later:

- Any non-success return values from `vkAcquireNextImageKHR()` are currently
  only logged but not handled. Instead the application is either supposed
  to re-create the swapchain resources or skip rendering and presentation.
  Since I couldn't coerce my Kubuntu setup to ever return a non-success value
  from `vkAcquireNextImageKHR()` I would have to implement behaviour I couldn't
  test, so I had to skip this part for now. Maybe when moving the code over
  to my Windows/NVIDIA PC I'll be able to handle that situation properly.
- Currently the swapchain image size must match the window client rectangle
  size (same as OpenGL via GLX). The Vulkan swapchain interface has an
  optional scaling feature, but I couldn't get this to work on my Kubuntu
  laptop. Window-system scaling is mainly useful when the system has a
  high-dpi display but lower-end GPU, and all other sokol-app backends
  depend on the system to scale a smaller framebuffer to the window client
  rectangle when needed.

The main area where I struggled with in the sokol-app Vulkan backend was
swapchain resizing. Most sokol-app backends kick off any swapchain
resize operation from the window system's resize event, e.g.:

- window is resized by user
- system resize event fires giving the new window size
- sokol-app resize event handler initiated a swapchain resize, records
  the new size (which is returned by `sapp_width/height()`) and
  notifies the app code via `SAPP_EVENTTYPE_RESIZED`

This didn't work on the Vulkan backend, the validation layer would sometimes
complain that there's a difference between actual and expected swapchain
surface dimensions (I forgot the exact error circumstances, forgiveable since
implementating a Vulkan backend is basically from crawling from one validation
layer error to the next).

Long story short: I got it to work by leaving the host window system
entirely out of the loop for resizing and let the Vulkan swapchain
take full control of the resize process:

- window is resized by user
- system resize event fires, but is now ignored by sokol-app
- the next time `vkQueuePresentKHR()` is called it returns with an error code
  which triggers a swapchain-resource re-creation (getting the size from the
  Vulkan surface object instead of the window system), records the new size
  (for `sapp_width/height()`) and finally fires an `SAPP_EVENTTYPE_RESIZED` event

This fixes any validation layer warnings and is in the end a cleaner
implementation compared to letting the window system dictate the swapchain size.

There are downsides though: At least on my Kubuntu laptop it looks like the
window system and Vulkan swapchain code doesn't run in in lock step. Instead the
Vulkan swapchain seems to lag behind the window system a bit and this results in
minor artefacts during resizing: sometimes there's a visible gap between the
Vulkan surface and window border, and the frame rate gets slighly out of whack
during resize. On macOS, Metal rendering during window resize is buttery smooth
and without resize-jitter or border-gaps (although tbf, removing the resize-jitter
on macOS took some effort).

That's all there is to the Vulkan backend in sokol_app.h, on to sokol_gfx.h!











#### FIXME: shorten this and turn from rant into constructive criticism

Also to get that out of the way: I still don't think that Vulkan is a
well-designed 3D-API, it doesn't feel like a 'designed' API at all, but rather like
an adhoc collection of concepts thrown together without any consideration
for how well those different concepts work together. From that perspective,
Vulkan (even without extensions) is already in similar messy state as
OpenGL (with the difference that it took Vulkan only a decade to reach that state).

IMHO it would be better if instead of Vulkan 1.4 we'd be at Vulkan 4.0 now, with
the assumption that each major version is breaking backward compatibility
(similar to how D3D versioning worked before D3D12 - btw, where's D3D13 and with
removal of deprecated cruft, extensions that had been merged into the core D3D14
Microsoft - they're both long overdue!). Such major breaking API versions which
remove extensions that had been incorporated into the core API and also remove
support for API features that appease outdated GPU architectures would make it
*much* easier to work with Vulkan for somebody who hasn't closely followed
each API change since 2016 (which is the exact same problem that OpenGL suffers from).

For example, the Vulkan implementation on my laptop currently exposes 177(!)
extensions, many of those are redundant because they had been incorporated into
the core API in minor Vulkan versions. I really don't understand why such
'redundant' extensions are listed when I'm specifically asking for a specific
version of the Vulkan API anyway - and the same should be true for API features
and type declarations. For instance why does the Vulkan header expose synchronization1
structs and function prototypes when I'm using synchronization2? At the least
let me define the intended Vulkan API version before including `<vulkan.h>` and
don't include the deprecated cruft from older API versions. Such 'version
scoping' would immediately improve the developer experience dramatically without
having to change the entire versioning philosophy.

IMHO Vulkan also has fundamentally failed at the 'low-level explicit API' promise.
GPUs still have significantly different resource binding models, and the idea
that GPU architectures would somehow converge to an ideal common architecture
hasn't manifested itself in the real-world a decade later and this fundamentally
clashes with the abstraction level of Vulkan. On one hand, GPU architectures
still fundamentally differ, but on the other hand the Vulkan API model is so
low-level that it cannot provide enough 'wiggle room' for drivers to implement
an optimal solution for a specific GPU. The result is a mess of optional
extensions which allow a more optimal code path on a specific GPU architecture
but either are not supported at all, or with reduced performance on other GPU
architectures. The result is that Vulkan could just as well be a bunch of
GPU-specific APIs, because a 'perfect' Vulkan-based rendering engine would need
to implement slightly different code paths depending on the underlying GPU
feature set - and once we're there, what's even the point of a common
'standard API'? IMHO Vulkan should just be honest and mark API features with a
GPU vendor 'nutrition score' - there's a shitton of 'best practices' posts by
GPU vendors which describe the best Vulkan code paths on their specific GPU architecture,
but this should really be centralized in the specification. And even if the result
is a handful completely different sub-APIs for resource binding, so be it - at least then
it would be easier to pick one instead of having to rely on hear-say which
extensions or API features provide the best performance on a specific GPU
architecture but are a bad choice on another specific GPU architecture.

I think all of the above is a 'Khronos organization syndrome', and in hindsight
it was foolish to expect that an OpenGL successor could somehow escape the same
fate as OpenGL. OpenGL didn't start as a bad 3D API, it only became one after
decades of deliberate work of the Khronos group ;)

There *are* some notable improvements over OpenGL though: everything is
better documented, the official sample code is in much better shape, and the
validation layer is very detailed and synchronized with the specification (e.g.
each validation layer message has a unique id which I can look up in the
specification - the specification itself is also more 'human readable' than the
OpenGL spec). As a result, writing code for the Vulkan API is much less
trial-and-error than writing code for OpenGL. The level of tediousness is about
the same though - it's not like OpenGL is much less verbose than Vulkan - the line count
of the sokol-gfx OpenGL backend is about the same as the Vulkan backend, and both
are about 1/3 bigger than the other backends.

At least there's also a silver lining on the horizon though, but let's see if
anything of that comes to frution (e.g. fundamental API changes instead of
just throwing even more extensions into the mix):

[The Road to the Future](https://www.youtube.com/watch?v=NM-SzTHAKGo)

Ok, rant over, back to sokol-gfx :)
