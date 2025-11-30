---
layout: post
title: "Experimental Sokol Vulkan backend"
---

In a couple of days I will merge the first implementation of a sokol-gfx
Vulkan backend. Please consider this backend as 'experimental', it has
only received limited testing, has limited platform coverage and some
known shortcomings and feature gaps which I will address in followup
updates.

The currently known limitiations are:

- the entire code expects a 'desktop GPU feature set' and doesn't implement
  fallback paths for mobile GPUs
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
  most code should work without or only minimal changes)
- the Vulkan validation layer is silent on all sokol-samples (which try to cover
  most sokol-gfx features and their combined usage), and this includes the tricky
  optional synchronization2 validations (I'm pretty proud of that considering
  that most Vulkan samples I tried have sync-validation errors)
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
  if I've ever seen one). This is also the main reason why mobile GPUs had to be left out
  for now, and apparently descriptor buffers are also a poor match for NVIDIA
  GPUs. The plan here is to wait until Khronos completes work on a
  descriptor pool replacement which AFAIK will be a mix of descriptor buffers and
  D3D12-style descriptor heaps and then port the `EXT_descriptor_buffer` code
  over to that new resource binding API
- 'synchronization2' (not a drastic change from the original barrier model,
  I'm just listing it here for completeness)

Work on the Vulkan backend spans three sub-projects:

- `sokol-shdc`: added Vulkan-flavoured SPIRV output
- `sokol_app.h`: device creation, swapchain management and frame loop
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

The weird double compilation is a compromise to avoid large structural changes
to the sokol-shdc code base and make the Vulkan shader pipeline less of a
special case. Essentially, SPIRV is used as an intermediate format in the
first compile pass, and then as output bytecode format in the second pass.

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
merged to better harmonize with sokol-gfx, e.g. the following functions to
obtain backend-specific 3D API objects have been removed:

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
the sokol-gfx structs `sg_environment` and `sg_swapchain` (with the emphasis on
**conceptually**, you still need a mapping from the sokol-app structs and enums
to the sokol-gfx structs and enums, and this mapping is still peformed by the
sokol_glue.h header.

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
`sapp_vulkan_swapchain vulkan;` with Vulkan object pointers which are needed
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
    and presentation commands all on the same queue
- Next a logical `vkDevice` object is created with the following required
  features and extensions (with the exception of compressed texture formats
  which are optional):
  - a single queue for all commands
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
    - present-mode hardwired to `VK_PRESENT_MODE_FIFO_KHR`
    - composite-alpha hardwired to `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`
    - `VkImage` and `VkImageView` objects are obtained or created
      for the swapchain images, depth-stencil-buffer
      and optional MSAA surface
- Finally a couple of VkSemaphore objects are created for each
  stage in the 'swapchain-pipeline':
    - one `render_finished_semaphore` which signals that the GPU
      has finished rendering to a swapchain surface
    - one `present_complete_semaphore` which signals that presenting
      a swapchain surface has completed and is ready for reuse

At this point, the Vulkan specific code in sokol_app.h is at about 600
lines of code, which is a lot of boilerplate, but OTH is a lot less messy
than the combined OpenGL window system code for GLX, EGL, WGL or NSOpenGL
(yet still a lot more than the window system glue for the other backends).

The actually interesting stuff happens in the last two Vulkan backend functions:

The internal function `_sapp_vk_swapchain_next()` is a wrapper around
`vkAcquireNextImageKHR()` and obtains the next free swapchain image. The
function will also signal the associated `present_complete_semaphore`.

The last function in the sokol-app Vulkan backend is `_sapp_vk_present()`, this
is a wrapper for `vkQueuePresentKHR()`. The present operation uses the
`render_finished_semaphore` to make sure that presentation happens after the GPU
has finished rendering to the swapchain image. When the `vkQueuePresentKHR()`
function returns with `VK_ERROR_OUT_OF_DATE_KHR` or `VK_SUBOPTIMAL_KHR`, the
swapchain resources are recreated (this happens for instance when the window is
resized).

There's a couple of open todo points in the sokol-app Vulkan backend which
I'll take care of later:

- Any non-success return values from `vkAcquireNextImageKHR()` are currently
  only logged but not handled. Normally the application is either supposed
  to re-create the swapchain resources or skip rendering and presentation.
  Since I couldn't coerce my Kubuntu laptop to ever return a non-success value
  from `vkAcquireNextImageKHR()` I would have to implement behaviour I couldn't
  test, so I had to skip this part for now. Maybe when moving the code over
  to my Windows/NVIDIA PC I'll be able to handle that situation properly.
- Currently the swapchain image size must match the window client rectangle
  size (same as OpenGL via GLX). The Vulkan swapchain API has an
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
- sokol-app listens for the window system resize event and initiates
  a swapchain resize with the new size coming from the window system
  event, the stores the new size for sapp_width/height() and finally
  fires an `SAPP_EVENTTYPE_RESIZED` event

This doesn't work on the Vulkan backend, the validation layer would sometimes
complain that there's a difference between actual and expected swapchain
surface dimensions (I forgot the exact error circumstances, forgiveable since
implementating a Vulkan backend is basically crawling from one validation
layer error to the next).

Long story short: I got it to work by leaving the host window system entirely
out of the loop and let the Vulkan swapchain take full control of the resize
process:

- window is resized by user
- system resize event fires, but is now ignored by sokol-app
- the next time `vkQueuePresentKHR()` is called it returns with an error code
  and this triggers a swapchain-resource resize, with the size coming from
  the Vulkan surface object instead of the window system, finally an
  `SAPP_EVENTTYPE_RESIZED` event is fired

This fixes any validation layer warnings and is in the end a cleaner
implementation compared to letting the window system dictate the swapchain size.

There are downsides though: At least on my Kubuntu laptop it looks like the
window system and Vulkan swapchain code doesn't run in in lock step. Instead the
Vulkan swapchain seems to lag behind the window system a bit and this results in
minor artefacts during resizing: sometimes there's a visible gap between the
Vulkan surface and window border, and the frame rate gets slighly out of whack
during resize. In comparison, on macOS rendering with Metal during window resize
is buttery smooth and without resize-jitter or border-gaps (although tbf,
removing the resize-jitter on macOS had to be implemented by anchoring the
NSView object to a window border).

That's all there is to the Vulkan backend in sokol_app.h, on to sokol_gfx.h!

## sokol_gfx.h changes

For the most part, the actual mapping of the sokol-gfx functions to Vulkan
API function is very straightforward, e.g. the sokol-gfx API matches
surprisingly well to the Vulkan API. This is mainly thanks to using a couple
of modern Vulkan features and extensions:

- Dynamic rendering (e.g. `vkBeginRendering()/vkEndRendering()`) is a perfect match
  to sokol-gfx `sg_begin_pass()/sg_end_pass()`, this is not very surprising though
  because the dynamic rendering Vulkan API is basically a 'de-OOP-ed' version of the Metal
  render pass API.
- `EXT_descriptor_buffers` is an absolutely perfect match to sokol-gfx's
  `sg_apply_bindings()` call, and a 'pretty good' match for `sg_apply_uniforms()`

The main areas for future improvements are the barrier system and the staging
system, but let's not get ahead of ourselves.

### A 10000 foot view

Apart from the straight mapping of sokol-gfx API calls to Vulkan-API calls, the
Vulkan backend has to implement a couple of low-level subsystems. This isn't
all that unusual, other backends also have such subsystems, but the Vulkan
backend definitely has the most 'subsystem code'.

OTH some concepts of modern Vulkan are quite similar to WebGPU, Metal and even
D3D11 - and this conceptual overlap significantly simplified the Vulkan
backend implementation.

In one area the Vulkan backend has even more straightforward implementations than
some of the other backends, for instance the implementation of the resource binding
call `sg_apply_bindings` in the Vulkan backend is one of the most straightforward
of all backends and especially compared to the WebGPU backend. In Vulkan it's
literally just a bunch of memcpy's followed by a single Vulkan API call to
record an offset into the descriptor buffer (ok, it's actually a bit more
complicated because of the barrier system). Compared to that, the WebGPU backend
needs to use a 'hash-and-cache' approach for baked BindGroup objects, e.g.
calling `sg_apply_bindings()` may involve creating and destroying API objects.

The low-level subsystems in the sokol-gfx Vulkan backend are:

- a 'delete queue' system for delayed resource destruction
- the GPU memory allocation system (very rudimentary at the moment)
- the frame-sync system (e.g. ensuring that the CPU and GPU can work
  in parallel in typical render frames)
- the uniform update system
- the bindings update system
- two 'staging systems' for copying CPU-side data into GPU-side resources:
    - a 'copy' staging system
    - a 'stream' staging system
- the resource barrier system

Let's look at those one by one:

### The Delete Queue System

Vulkan doesn't have any automatic lifetime management like some other 3D APIs
(e.g. no D3D-style reference counting). When you call a destroy function on
an object, it's gone. When you do that while the object is still in flight
(e.g. referenced in a queue and waiting to be consumed by the GPU), hilarity
ensues.

IMHO this is much better than any automatic lifetime management system, because
it avoids any confusion about reference counts (e.g. questions like: when I call
this function to get a object, will that bump the refcount or not?), but this
means that a Vulkan backend needs to implement some sort of of garbage collection
on its own.

Sokol-gfx uses a double-buffered delete-queue system for this. Each
'double-buffer-frame-context' owns a delete queue which is a simple fixed-size
array of pointer-pairs. Each queue item consists of:

- one type-erased Vulkan object pointer (e.g. a simple void*)
- a function pointer for a destructor function which takes a
  void* as argument and knows how to destroy that Vulkan object

All Vulkan objects types which may be referenced in command buffers will
not call their `vkDestroy*()` functions directly, but instead add them
to the delete-queue that's associated with the currently recording command
buffer. At the start of a new frame (what 'new frame' actually means is
explained down in the 'frame-sync system'), the delete-queue for that
frame-context is drained by calling the destructor function with the
Vulkan objec pointer of a queue item.

### The GPU Memory Allocation System

Currently GPU allocations do *not* go through a custom allocator, instead
all granular allocation directly call into `vkAllocateMemory()`. Originally
I had intended to use SebAaltonen's [OffsetAllocator](https://github.com/sebbbi/OffsetAllocator)
as the default GPU allocator, but also expose an allocator interface to allow
users to hook in custom allocators like [VMA](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator).

Historically a custom allocator was pretty much required because some Vulkan
drivers only allowed 4096 unique GPU allocations. Today though it looks
like pretty much all (desktop) Vulkan drivers allow 4 billion allocations (at least
according to the [Vulkan hardware database](https://vulkan.gpuinfo.org/).

The plan is still to at least allow hooking up a custom GPU allocator via an
allocator interface, and also maybe to integrate OffsetAllocator as default
allocator, but without knowing the memory allocation strategy of Vulkan drivers
this may be redundant. E.g. if a Vulkan driver essentially integrates something
like VMA anyway there's not much point stacking another allocator on top of it,
at least with a fairly high level API wrapper like sokol-gfx.

In any case, the current GPU memory allocation implementation is prepared for
a bit more abstraction in the future. All GPU allocation go through a single
internal function `_sg_vk_mem_alloc_device_memory()` which takes a 'memory type'
enum and a `VkMemoryRequirements` pointer as input. The memory type enum is
sokol-gfx specific and includes:

- storage buffer (an sg_buffer object with storage buffer usage)
- generic buffer (all other sg_buffer types)
- an image (all usages)
- an internal staging buffer for the 'copy-staging system'
- an internal staging buffer for the 'stream-staging system'
- an internal uniform buffer
- an internal descriptor buffer

Currently all resources are either in 'device-local' memory, or in
'host-visible + host-coherent' memory. Having the mapping from sokol-specific
memory type to Vulkan memory flags in one place makes it easier to tweak those
flags in the future (or delegate that decision to an external memory allocator).

### The Frame Sync System

The frame sync system is mainly concerned about letting the CPU and GPU work
in parallel without stepping on each other's feet. This basically comes down
to double-buffer all resources which are written by the CPU and read by the
GPU, and to have one sync-point in a sokol-gfx frame where the CPU needs to
wait for the oldest 'frame-context' to become available (e.g. is no longer
read by the GPU, i.e. is 'no longer in flight').

This single `CPU <=> GPU` sync point is implemented in a function
`_sg_vk_acquire_frame_command_buffers()`.  The name indicates the main feature
of that function: it acquires command buffers to record the Vulkan commands of
the current frame. Command buffers are reused, so this involves waiting for the
commands buffers to become available (e.g. they are no longer read from by the
GPU).

For this `CPU <=> GPU` synchronization, each double-buffered frame-context owns
a `VkFence` which is signalled when the GPU is done processing a 'submit' (more on that later).

So the first and most important thing the `_sg_vk_acquire_frame_command_buffers()` function
does is to wait for the fence of the oldest frame-context with a call to `vkWaitForFences()`.

This wait-operation is the reason why sokol-gfx applications should move any sokol-gfx
towards the end of a frame and try do all the heavy CPU works before the first sokol-gfx
call. Specifically calls to:

- `sg_begin_pass()`
- `sg_update_buffer()`
- `sg_update_image()`
- `sg_append_buffer()`

...these are basically the 'potential new-frame entry points' of the sokol-gfx
API which may require the CPU to wait for the GPU.

The `_sg_vk_acquire_frame_command_buffers()` function does a couple more things
after `vkWaitForFences()` returns:

- first it checks if the function had already been called in the current frame,
  if yes it returns immediately
- `vkResetFences()` is called on the fence we just waited on
- the delete-queue is drained (e.g. all resources which were recorded for
  destruction in the frame-context we just waited on are finally destroyed)
- any command buffers associated with the new frame are reset via `vkResetCommandBuffer()`
- ...and recording into those command buffers is started via `vkBeginCommandBuffer()`
- additionally the other subsystems are informed because they might want to do their
  own thing by calling:
  - `_sg_vk_uniform_after_acquire()`
  - `_sg_vk_bind_after_acquire()`
  - `_sg_vk_staging_stream_after_acquire()`

The other internal function of the frame-sync system is `_sg_vk_submit_frame_command_buffers()`.
This is called at the end of a 'sokol-gfx frame' in `sg_commit()` call. The main job
of this function is to call submit the recorded command buffers for the current frame
via `vkQueueSubmit()`. This submit operation uses the two semaphores we got handed
from the outside world (e.g. sokol-app) as part of the swapchain information
in `sg_begin_pass()`:

- the `present_complete_semaphore` is used as the wait-semaphore of the
  `vkQueueSubmit()` call (the GPU basically needs to wait for the swapchain image
  of the render pass to become available for reuse)
- the `render_finished_semaphore` is used as the signal-semaphore to be signalled
  when the GPU is done processing the submit payload

Before the `vkQueueSubmit()` call there's a bit more housekeeping happening:

- the other subsystems are notified about the submit via:
    - `_sg_vk_staging_stream_before_submit()`
    - `_sg_vk_bind_before_submit()`
    - `_sg_vk_uniform_before_submit()`
- recording into the command buffers which are associated with the current
  frame context are is finished via `vkEndCommandBuffers()`

It's also important to note that there is one other potential `CPU <=> GPU`
sync-point in a frame, and that's in the first `sg_begin_pass()` for a
swapchain render pass: the swapchain-info struct that's passed into
`sg_begin_pass()` contains a swapchain image which must be acquired via
`vkAcquireNextImageKHR()` (when using sokol_app.h this happens in the
`sapp_get_swapchain()` call - usually indirectly via `sglue_swapchain()`).

That is all for frame-sync system in sokol-gfx, all in all quite similar to
Metal or WebGPU, just with more code bloat (as is the Vulkan way).

### Resource binding via EXT_descriptor_buffer

...a little detour into Vulkan descriptors and how the Vulkan sokol-gfx maps
its resource binding model to Vulkan.

Conceptually and somewhat simplified, a Vulkan **descriptor** is an abstract
reference to a Vulkan buffer, image or sampler which needs to be accessible in a
shader. Basically what shows up on the shader side whenever you see a
`layout(binding=x) ...`. In sokol-gfx lingo this is called a 'binding'.

In an ideal world, such a binding is simply a 'GPU pointer' to some
opaque struct living in GPU memory which describes to shader code how
to access bytes in a storage buffer, pixels in an storage image, or how
to perform a texture-sampling operation).

In the real world it's not that simple because this is exactly the one main area
where GPU architectures differ the most: on some GPUs this information is
hardwired into register tables and/or involves fixed-function features instead
being just 'structs in GPU memory' - and unfortunately those differences are
not limited shitty mobile GPUs, but are also still present in desktop GPUs.
Intel, AMD and NVIDIA all have different opinions on how this whole resource binding
thing should work - and I'm not sure anything has changed in the last decade
since Vulkan promised us a more-or-less direct mapping to the underlying hardware.

So in the real world 3D APIs still need to come up with some sort of abstraction
layer to get all those different hardware resource binding models under a common
programming model (and yes, even the apparently 'low-level' Vulkan API had to
come up with a highlevel abstraction for resource binding - and this went quite
poorly... but I disgress).

(side note: traditional vertex- and index-buffer-bindings are *not* performed
through Vulkan descriptors, but through regular 'bindslot-setter' calls like in
any other 3D API - go figure).

A Vulkan **descriptor-set** is a group of such concrete bindings which can be
applied as an atomic unit instead of applying each descriptor individually. In
the end the traditional Vulkan descriptor model isn't all that different from
the 'old' bindslot model used in Metal V1 or D3D11, the one big and important
difference is that bindings are no longer applied individually but as groups.

The downside of such a 'bind group model' is course that specific combinations
may be unpredictable - which is the one big recurring topic in Vulkan (very slow)
API evolution.

In 'old Vulkan' pretty much all state-combinations in all areas of the API need
to be known upfront in order to move as much work as possible into the
init-phase and out of the render-phase. Theoretically a pretty sensible plan,
but unfortunately only theoretically. In practice there are a lot of use cases
where pre-baking everything is simply not possible, especially outside the game
engine world, and even in gaming it doesn't quite work - whenever you see
stuttering when something new appears on screen in modern games built on top of
state-of-the-art engines calling into modern 3D APIs -that's the basic design
philosophy of Vulkan and D3D12 crashing and burning after colliding with
reality.

Thankfully - but unfortunately very slowly - this is changing. Most of Vulkan's
progress in the last decade was about rolling the API back to a more dynamic
programming model. But sometimes this is going too far. OpenGL's flat 'state soup'
was equally bad, not necessarily for performance reason but because of robustness.
In OpenGL it is very simple to mess up the internal state machine by accidentially
leaking incorrect state from the last draw call or even the last frame.

IMHO the ideal middle ground is small-ish immutable state objects like in D3D11
or Metal. Immutability in small doses is actually a very good thing!

...anyway, back to Vulkan descriptors...

A Vulkan **descriptor-set-layout** is the *shape* of a descriptor-set, think of
it as the descriptor-sets type which needs to be known upfront before any
concrete descriptor-sets can exist. It basically says 'there will be a sampled
texture at binding 0, a buffer at binding 1 and a sampler at binding 2', but not
the concrete texture, buffer or sampler objects (those are recorded into a
concrete descriptor-set).

And finally a Vulkan **pipeline-layout** groups all descriptor-set-layouts required
by a the shaders of a Vulkan pipeline-state-object.

When coming from WebGPU this should all sound quite familiar since the
WebGPU bindgroups model is essentially the Vulkan 1.0 descriptor model
(for better or worse):

- WebGPU BindGroupEntry maps to Vulkan descriptors
- WebGPU BindGroup maps to Vulkan descriptor sets
- WebGPU BindGroupLayout maps to Vulkan descriptor set layouts
- WebGPU PipelineLayout maps to Vulkan pipeline layouts

Traditional Vulkan then adds descriptor pools on top of that but tbh I didn't
even bother to deal with those and skipped right to `EXT_descriptor_buffer`.

With the descriptor buffer extension, descriptors and descriptor sets are 'just
memory' with opaque memory layouts for each descriptor type which are
specific to the Vulkan driver (depending on the driver and descriptor type, such
opaque memory blobs seem to be between 16 and 256 bytes per descriptor).

Binding resources with `EXT_descriptor_buffers` essentially looks like this:

- create a descriptor buffer big enough to hold all descriptors needed in
  a worst-case frame
- for all bindings in a descriptor-set-layout, ask Vulkan for the size and
  relative offsets (to the start of the descriptors) and store those somewhere
- similar for all concrete descriptors, ask Vulkan to copy their opaque memory
  representation into some memory location and keep those around for later
- then during rendering, simply write descriptor sets by memcpy'ing the opaque
  descriptor blobs into the descriptor buffer, using the offsets we also got
  upfront
- finally record the offset into the descriptor buffer into a Vulkan command
  buffer - and that's it

This is pretty much the same procedure how uniform data snippet updates are
performed in the sokol-gfx Metal and WebGPU backends, now just extended to
resource bindings.

E.g. TL;DR: both uniform data snippets and resource bindings are
'just frame-transient data snippets' which are memcpy'ed into per-frame
buffers and the buffer offsets recorded before the next draw- or dispatch-call.

In sokol-gfx all Vulkan objects required for the resource binding system can be
created upfront:

- descriptor-set-layouts and pipeline-layouts are created in `sg_make_shader()`
  using the shader interface reflection information in `sg_shader_desc`
  (uniform block bindings live in descriptor set 0, and texture, storage buffer
  storage image and sampler bindings live in descriptor set 1)
- the `EXT_descriptor_buffer` specific descriptor sizes and offsets are also
  queried and stored in `sg_make_shader()`
- the per-descriptor opaque memory blobs are copied upfront into sokol-gfx
  view objects during `sg_make_view()` (the only exception here are uniform
  block descriptors because those have an unpredictable dynamic offset)

...so a 'resource binding' operation in the sokol-gfx Vulkan backend via
the `EXT_descriptor_buffer` extension essentially comes down to:

- copy a bunch of small-ish (16..256 bytes) memory blocks into the
  current frame's descriptor buffer
- call a Vulkan function to record an offset into the descriptor buffer
  for the next draw- or dispatch-call


### The uniform update system:

Conceptually uniform updates in the Vulkan backned are similar to the Metal backend:

- a double-buffered uniform buffer big enough to hold all uniform updates
  for a worst-case frame, allocated in host-visible memory (so that the memory
  is directly writable by the CPU and directly readable by the GPU)
- a call to `sg_apply_uniforms()` memcpy's the uniform data snippet into the
  next free uniform buffer location (taking alignment requirements into account),
  this can happen independently for up to 8 'uniform block slots'
- before the next draw- or dispatch-call, the offsets into the uniform buffer for the up to
  8 uniform block slots are recorded into the current command buffer

The last step of recording the uniform-buffer offsets is delayed into the next
draw- or dispatch-call to avoid redundant work. This is because `sg_apply_uniforms()`
works on a single uniform block slot, but in Vulkan all uniform block slots are
grouped into one descriptor set, and we only want to apply that descriptor-set
at most once per draw/dispatch call.

The actual `sg_apply_uniforms()` call is extremely cheap:

- a simple memcpy of the uniform data snippet into the per-frame uniform buffer
- writing the 'GPU buffer address' and snippet size into a cached
  array of `VkDescriptorAddressInfoEXT` structs
- setting a 'uniforms dirty flag'.

...then later in the next draw- or dispatch-calls if the 'uniforms dirty flag'
is set the actual uniform block descriptor set binding happens:

- for each uniform block used in the current pipeline/shader, a opaque descriptor
  memory blob is directly written into the frame's descriptor buffer via
  a call to `vkGetDescriptorEXT()`
- the start offset of the descriptor-set in the descriptor buffer is recorded
  into the current frame command buffer via `vkCmdSetDescriptorBufferOffsetsEXT()`

...delaying the binding operation for uniform data into the draw- or dispatch-call
to avoid redundant calls is actually something that I'll also do in the WebGPU
backend (I was taking notes while implementing the Vulkan backend which improvements
could be back-ported to the WebGPU backend, and I'll take care of those
right after the Vulkan backend is merged).

### The resource binding system

(TODO)


### The copy-staging system

(TODO)

### The stream-staging system

(TODO)

### Everything else...

(TODO)




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
