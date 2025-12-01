---
layout: post
title: "The experimental Sokol Vulkan backend"
---

In a couple of days I will merge the first implementation of a sokol-gfx
Vulkan backend. Please consider this backend as 'experimental', it has
only received limited testing, has limited platform coverage and some
known shortcomings and feature gaps which I will address in followup
updates.

The related PRs are here:

- [sokol/#1350](https://github.com/floooh/sokol/pull/1350) - this one also
  has all the embedded shaders for the sokol 'utility headers', so it looks much
  bigger than it actually is (the sokol-gfx backend is around the same size as
  the GL backend, a bit over 3 kloc)
- [sokol-tools/#196](https://github.com/floooh/sokol-tools/pull/196) - this
  is the update for the shader compiler which is already merged

The currently known limitiations are:

- the entire code expects a 'desktop GPU feature set' and doesn't implement
  fallback paths for mobile or generally ancient GPUs
- the window system glue in sokol_app.h is only implemented for Linux/X11 - and before the question comes up again: it works just fine on Wayland-only distros
- only tested on an Intel Meteor Lake integrated GPU (which also means
  that some buffer types may be allocated in memory types that are not
  optimal on GPUs without unified memory)
- barriers for CPU => GPU updates are currently quite conservative
  (e.g. more barriers might be inserted than needed, or at a too
  early point in a frame)
- there's currently no GPU memory allocator, nor a way to inject
  an external GPU memory allocator like VMA (at least the latter is planned)
- rendering is currently only supported to a single swapchain
  (not a problem when used with sokol_app.h because that also only
  supports a single window)
- it's currently not possible to inject native Vulkan buffers and images
  into sokol-gfx (that's a somewhat esoteric feature supported by
  the other backends)
- I couldn't get RenderDoc to work, but it's unclear why

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
OpenGL backend (first, an alternative to WebGL2 had to happen - which exists now
with WebGPU, and next an alternative for OpenGL on Linux (and less important:
Android) had to be implemented (which is the Vulkan backend). So far Linux and
Android were the only sokol-gfx target platforms limited to a single backend: OpenGL.
All other target platforms already have a more modern alternative (Windows with
D3D11 and macOS/iOS with Metal). Deprecating the OpenGL backend won't happen for
a while, but personally I can't wait to free sokol-gfx from the 'shackles of
OpenGL' ;)

Also another reason why I felt that now is the right time to tackle Vulkan support
is that the Vulkan API has improved quite a bit since 1.0 in ways that make it a much
better fit for sokol-gfx. In a nutshell (if you already know Vulkan concepts),
the sokol-gfx backend makes use of the following 'modern' Vulkan features:

- 'dynamic rendering' (e.g. render passes are enclosed by begin/end
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
merged to better harmonize with sokol-gfx:

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

...those have been merged into:

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
  swapchain image (the number of swapchain images is essentially
  dictated by the Vulkan driver):
    - one `render_finished_semaphore` which signals that the GPU
      has finished rendering to a swapchain surface
    - one `present_complete_semaphore` which signals that presenting
      a swapchain image has completed and the image ready for reuse

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

The main area I struggled with in the sokol-app Vulkan backend was
swapchain resizing. Most sokol-app backends kick off any swapchain
resize operation from the window system's resize event, e.g.:

- window is resized by user
- window system resize event fires giving the new window size
- sokol-app listens for the window system resize event and initiates
  a swapchain resize with the new size coming from the window system
  event, then stores the new size for sapp_width/height() and finally
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
- window system resize event fires, but is now ignored by sokol-app
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
removing the resize-jitter on macOS had to be explicitly implemented by
anchoring the NSView object to a window border).

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

In some areas the Vulkan backend has even more straightforward implementations than
some of the other backends, for instance the implementation of the resource binding
call `sg_apply_bindings` in the Vulkan backend is one of the most straightforward
of all backends and especially compared to the WebGPU backend. In Vulkan it's
literally just a bunch of memcpy's followed by a single Vulkan API call to
record an offset into the descriptor buffer (ok, it's actually a bit more
complicated because of the barrier system). Compared to that, the WebGPU backend
needs to use a 'hash-and-cache' approach for baked BindGroup objects, e.g.
calling `sg_apply_bindings()` may involve creating and destroying WebGPU objects.

The low-level subsystems in the sokol-gfx Vulkan backend are:

- a 'delete queue' system for delayed Vulkan object destruction
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
this function to get an object reference, will that bump the refcount or not?),
but this means that a Vulkan backend needs to implement some sort of of garbage
collection on its own.

Sokol-gfx uses a double-buffered delete-queue system for this. Each
'double-buffer-frame-context' owns a delete queue which is a simple fixed-size
array of pointer-pairs. Each queue item consists of:

- one type-erased Vulkan object pointer (e.g. a void-pointer)
- a function pointer for a destructor function which takes a
  void* as argument and knows how to destroy that Vulkan object

All Vulkan objects types which may be referenced in command buffers will not
call their `vkDestroy*()` functions directly, but instead add them to the
delete-queue that's associated with the currently recorded command buffer. At
the start of a new frame (what 'new frame' actually means is explained down in
the 'frame-sync system'), the delete-queue for that frame-context is drained by
calling the destructor function with the Vulkan object pointer of a queue item.
This makes sure that any Vulkan objects are kept alive until the GPU has finished
processing any command buffers which might hold references to those objects.

### The GPU Memory Allocation System

Currently GPU allocations do *not* go through a custom allocator, instead
all granular allocations directly call into `vkAllocateMemory()`. Originally
I had intended to use SebAaltonen's [OffsetAllocator](https://github.com/sebbbi/OffsetAllocator)
as the default GPU allocator, but also expose an allocator interface to allow
users to inject more complex allocators like [VMA](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator).

Historically a custom allocator was pretty much required because some Vulkan
drivers only allowed 4096 unique GPU allocations. Today though it looks
like pretty much all (desktop) Vulkan drivers allow 4 billion allocations (at least
according to the [Vulkan hardware database](https://vulkan.gpuinfo.org/)).

The plan is still to at least allow injecting a custom GPU allocator via an
allocator interface, and also maybe to integrate OffsetAllocator as default
allocator, but without knowing the memory allocation strategy of Vulkan drivers
this may be redundant. E.g. if a Vulkan driver essentially integrates something
like VMA anyway there's not much point stacking another allocator on top of it,
at least for a fairly high level API wrapper like sokol-gfx.

In any case, the current GPU memory allocation implementation is prepared for
a bit more abstraction in the future. All GPU allocations go through a single
internal function `_sg_vk_mem_alloc_device_memory()` which takes a 'memory type'
enum and a `VkMemoryRequirements` pointer as input. The memory type enum is
sokol-gfx specific and includes:

- a storage buffer (an sg_buffer object with storage buffer usage)
- a generic buffer (all other sg_buffer types)
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
to double-buffering all resources which are written by the CPU and read by the
GPU, and to have one sync-point in a sokol-gfx frame where the CPU needs to
wait for the oldest 'frame-context' to become available (e.g. is no longer
'in flight').

This single `CPU <=> GPU` sync point is implemented in a function
`_sg_vk_acquire_frame_command_buffers()`.  The name indicates the main feature
of that function: it acquires command buffers to record the Vulkan commands of
the current frame. Command buffers are reused, so this involves waiting for the
commands buffers to become available (e.g. they are no longer read from by the
GPU). "Command buffers" is plural because there are two command buffers per frame:
one which records all staging-commands, and one for the actual compute/render
commands - more on that later in the staging system section.

For this `CPU <=> GPU` synchronization, each double-buffered frame-context owns
a `VkFence` which is signalled when the GPU is done processing a 'queue submit'.

So the first and most important thing the `_sg_vk_acquire_frame_command_buffers()` function
does is to wait for the fence of the oldest frame-context with a call to `vkWaitForFences()`.

This potential-wait-operation is the reason why sokol-gfx applications should move
sokol-gfx code towards the end of the frame callback and try to do all
heavy non-rendering-related CPU work at the start of the frame callback.
More specifically calls to:

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
  own thing:
  - `_sg_vk_uniform_after_acquire()`
  - `_sg_vk_bind_after_acquire()`
  - `_sg_vk_staging_stream_after_acquire()`

The other internal function of the frame-sync system is `_sg_vk_submit_frame_command_buffers()`.
This is called at the end of a 'sokol-gfx frame' in the `sg_commit()` call. The main job
of this function is to submit the recorded command buffers for the current frame
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
  frame context is finished via `vkEndCommandBuffers()`

It's also important to note that there is one other potential `CPU <=> GPU`
sync-point in a frame, and that's in the first `sg_begin_pass()` for a
swapchain render pass: the swapchain-info struct that's passed into
`sg_begin_pass()` contains a swapchain image which must be acquired via
`vkAcquireNextImageKHR()` (when using sokol_app.h this happens in the
`sapp_get_swapchain()` call - usually indirectly via `sglue_swapchain()`).

That is all for the frame-sync system in sokol-gfx, all in all quite similar to
Metal or WebGPU, just with more code bloat (as is the Vulkan way).

### Resource binding via EXT_descriptor_buffer

...a little detour into Vulkan descriptors and how the sokol-gfx resource binding
model maps to Vulkan.

Conceptually and somewhat simplified, a Vulkan **descriptor** is an abstract
reference to a Vulkan buffer, image or sampler which needs to be accessible in a
shader. Basically what shows up on the shader side whenever you see a
`layout(binding=x) ...`. In sokol-gfx lingo this is called a 'binding'.

In an ideal world, such a binding would simply be a 'GPU pointer' to some
opaque struct living in GPU memory which describes to shader code how
to access bytes in a storage buffer, pixels in an storage image, or how
to perform a texture-sampling operation).

In the real world it's not that simple because this is exactly the one main area
where GPU architectures still differ dramatically: on some GPUs this information might be
hardwired into register tables and/or involves fixed-function features instead
of being just 'structs in GPU memory' - and unfortunately those differences are
not limited to shitty mobile GPUs, but are also still present in desktop GPUs.
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
applied as an atomic unit instead of applying each binding individually. In
the end the traditional Vulkan descriptor model isn't all that different from
the 'old' bindslot model used in Metal V1 or D3D11, the one big and important
difference is that bindings are not applied individually but as groups.

The downside of such a 'bind group model' is of course that specific binding
combinations may be unpredictable - which is the one big recurring topic in
Vulkan's (very slow) API evolution.

In 'old Vulkan' pretty much all state-combinations in all areas of the API need
to be known upfront in order to move as much work as possible into the
init-phase and out of the render-phase. Theoretically a pretty sensible plan,
but unfortunately only theoretically. In practice there are a lot of use cases
where pre-baking everything is simply not possible, especially outside the game
engine world, and even in gaming it doesn't quite work - whenever you see
stuttering when something new appears on screen in modern games built on top of
state-of-the-art engines calling into modern 3D APIs - that's most likely the core design
philosophy of Vulkan and D3D12 crashing and burning after colliding with
reality. Thankfully - but unfortunately very slowly - this is changing. Most of Vulkan's
progress in the last decade was about rolling the core API back to a more 'dynamic'
programming model.

Ok, back to Vulkan's resource binding lingo:

A Vulkan **descriptor-set-layout** is the *shape* of a descriptor-set.
It basically says 'there will be a sampled texture at binding 0, a buffer at
binding 1 and a sampler at binding 2', but not the concrete texture, buffer or
sampler objects (those are referenced in the concrete **descriptor-sets**).

And finally a Vulkan **pipeline-layout** groups all descriptor-set-layouts required
by the shader stages of a Vulkan pipeline-state-object.

When coming from WebGPU this should all sound quite familiar since the
WebGPU bindgroups model is essentially the Vulkan 1.0 descriptor model
(for better or worse):

- WebGPU BindGroupEntry maps to Vulkan descriptors
- WebGPU BindGroup maps to Vulkan descriptor sets
- WebGPU BindGroupLayout maps to Vulkan descriptor set layouts
- WebGPU PipelineLayout maps to Vulkan pipeline layouts

'Old Vulkan' then adds descriptor pools on top of that but tbh I didn't
even bother to deal with those and skipped right to `EXT_descriptor_buffer`.

With the descriptor buffer extension, descriptors and descriptor sets are 'just
memory' with opaque memory layouts for each descriptor type which are
specific to the Vulkan driver (depending on the driver and descriptor type, such
opaque memory blobs seem to be between 16 and 256 bytes per descriptor).

Binding resources with `EXT_descriptor_buffers` essentially looks like this:

In the init-phase:
- create a descriptor buffer big enough to hold all descriptors needed in
  a worst-case frame
- for each item in a descriptor-set-layout, ask Vulkan for the descriptor size and
  relative offset to the start of the descriptor-set data in the descriptor buffer
- similar for all concrete descriptors, ask Vulkan to copy their opaque memory
  representation into some private memory location and keep those around for the render
  phase (of course it's also possible to move this step into the render phase)

In the render-phase:
- memcpy the concrete descriptor set blobs we stored upfront into
  the descriptor buffer, using the offsets we also stored upfront
- finally record the start offset in the descriptor buffer into a Vulkan command
  buffer via a Vulkan API call, and that's it!

This is pretty much the same procedure how uniform data updates are
performed in the sokol-gfx Metal and WebGPU backends, now just extended to
resource bindings.

E.g. TL;DR: both uniform data snippets and resource bindings are
'just frame-transient data snippets' which are memcpy'ed into per-frame
buffers and the buffer offsets recorded before the next draw- or dispatch-call.

In sokol-gfx, the VkDescriptorSetLayout and VkPipelineLayout objects are created
in `sg_make_shader()` using the shader interface reflection information provided
in `sg_shader_desc` arg (which is usually code-generated by the sokol-shdc
shader compiler).

- the first descriptor set layout (set 0) describes all uniform block bindings
  used by the shader across all shader stages
- the second descriptor set layout (set 1) describes all texture, storage buffer,
  storage image and sampler bindings

...additionally, `sg_make_shader()` queries the descriptor sizes and offsets
within their descriptor set (this is `EXT_descriptor_buffer` functionality).

### The uniform update system:

Conceptually uniform updates in the Vulkan backend are similar to the Metal backend:

- a double-buffered uniform buffer big enough to hold all uniform updates
  for a worst-case frame, allocated in host-visible memory (so that the memory
  is directly writable by the CPU and directly readable by the GPU)
- a call to `sg_apply_uniforms()` memcpy's the uniform data snippet into the
  next free uniform buffer location (taking alignment requirements into account),
  this happens individually for the up to 8 'uniform block slots'
- before the next draw- or dispatch-call, the offsets into the uniform buffer for the up to
  8 uniform block slots are recorded into the current command buffer

The last step of recording the uniform-buffer offsets is delayed into the next
draw- or dispatch-call to avoid redundant work. This is because `sg_apply_uniforms()`
works on a single uniform block slot, but in Vulkan all uniform block slots are
grouped into one descriptor set, and we only want to apply that descriptor-set
at most once per draw/dispatch call.

The actual `sg_apply_uniforms()` call is extremely cheap since no Vulkan API
calls are performed:

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

...delaying the operation to record the uniform buffer offsets into the draw- or
dispatch-call to avoid redundant API calls is actually something that I will also
need to implement in the WebGPU backend (I was taking notes while implementing
the Vulkan backend which improvements could be back-ported to the WebGPU
backend, and I'll take care of those right after the Vulkan backend is merged).

### The resource binding system

Updating resource bindings via `sg_apply_bindings()` is very similar to the
uniform update system, but actually even simpler because no extra uniform buffer
is involved, and some more initialization can be moved into the init-phase when
creating view objects:

When creating a texture-, storage-buffer- or storage-image-view object via
`sg_make_view()` or a sampler object via `sg_make_sampler)`, the concrete
descriptor data (those little 16..256 byte opaque memory blobs) is copied into
the sokol-gfx view or sampler object via `vkGetDescriptorEXT()`.

Then `sg_apply_bindings()` is just a couple of memcpy's and a Vulkan call:

- for each view and sampler in the `sg_bindings` argument, a memcpy of the
  descriptor memory blob which was stored in the sokol-gfx view object
  into the current frame's descriptor buffer happens - e.g. no
  Vulkan calls for that...
- finally a single call to `vkCmdSetDescriptorBufferOffsetsEXT()` records
  the descriptor buffer offset into the current frame's command buffer

Vertex- and index-buffer bindings happen via traditional bindslot calls
(`vkCmdBindVertexBuffers` and `vkCmdBindIndexBuffer`). Additionally,
barriers may be inserted inside `sg_apply_bindings()` but that will be explained
further down in the barrier system.

### The two staging systems

Sokol-gfx currently has two separate staging systems for uploading CPU-side
data into GPU-memory with the rather arbitrary names 'copy-staging-system' and
'stream-staging-system'. Both can upload data into buffers and images, but with
different compromises:

- the 'copy-staging-system' can upload large amounts of data through a single
  small staging buffer (default size: 4 MB), with the downside that the Vulkan
  queue needs to be flushed (e.g. a `vkQueueWaitIdle()` is involved)
- the 'stream-staging-system' can upload a limited amount of data per-frame
  through a fixed-size double-buffer staging buffer (default size: 16 MB -
  but this can be tweaked in the `sg_setup()` call of course), this doesn't cause
  any frame-pacing 'disruptions' like the copy-staging-system does

The copy-staging-system is currently used:

1. to upload initial content into immutable buffers and images within
   `sg_make_buffer()` and `sg_make_image()`
2. to upload data into `usage.dynamic_update` images and buffers
   in the `sg_update_buffer()`, `sg_append_buffer()` and `sg_update_image()` calls

The stream-staging system is only used for `usage.stream_update` resources
when calling `sg_update_buffer()`, `sg_append_buffer()` and `sg_update_image()`.

This means that the correct choice of `usage.dynamic_update` and
`usage.stream_update` for buffers and images is much more important in the
Vulkan backend than in other backends.

In general:

- creating an immutable buffer or image **with initial content** in the
  render-phase will 'disrupt' rendering (how bad this disruption actually is
  remains to be seen though)
- the same disruption happens for updating a buffer or image with `usage.update_dynamic`,
- make sure to use `usage.stream_update` for buffers and images that need to be updated each
  frame, but be aware that those uploads go through a single per-frame staging
  buffer which needs to be big enough to hold all stream-uploads in a single
  frame (staging buffer sizes can be adjusted in the sg_setup() call)

The strategy for updating `usage.dynamic_update` may change in the future. For
instance I was considering treating dynamic-updates exactly the same as
stream-updates (e.g. going through the per-frame staging buffer to avoid
the `vkQueueWaitIdle()`), and when the staging buffer would overflow
fall back to the copy-staging system (also for stream-updates). This
felt too unpredictable to me, so I didn't go that way for now.

Note that the staging system is the most likely system to drastically change
in the future (together with the barrier system). One of the important planned
changes in my mental sokol-gfx roadmap is a rewrite of the resource update API,
and this rewrite will most likely 'favour' modern 3D APIs and not worry about
OpenGL as much as the current very restrictive resource update API does.

The common part in both staging systems is how the actual upload happens:

- staging buffers are allocated in CPU-visible + cache-coherent memory
  (the copy-staging system uses a single small buffer, while the stream-staging
  system uses double-buffering)
- a staging operations first memcpy's a chunk of memory into the staging
  buffer and then records a Vulkan command to copy that data from the
  staging buffer into a Vulkan buffer or image (via `vkCmdCopyBuffer` or
  `vkCmdCopyBufferToImage2()`
- in the stream-staging system each buffer update is always a single call
  to `vkCmdCopyBuffer()` and each image update is always one call to
  `vkCmdCopyBufferToImage2()` per mipmap
- in the copy-staging-system, staging operations which are bigger than the
  staging buffer size will be split into multiple copy operations,
  each copy-step involving a `vkQueueWaitIdle`
- overflowing the stream-staging buffer is a 'soft error', e.g. an
  error will be logged but otherwise this is a no-op

There is another notable implementation detail in the stream-staging
system which is related to the barrier system:

All stream-staging copy commands are recorded into a separate Vulkan command
buffer object so that they are not interleaved with the compute/render commands
which are recorded into the regular per-frame command buffer.

This is done to move any staging commands out of render passes which is pretty
much required for barrier management (I don't quite remember though if the
Vulkan validation layer only complained about issuing barriers inside
`vkBeginRendering/vkEndRendering` or if copy commands were also prohibited during
the render phase).

Long story short: all Vulkan commands used for staging operations are recorded
into a separate command buffer so that all GPU => CPU copies can be moved in
front of any computer/render commands because of various Vulkan API usage
restrictions. This was necessary because sokol-gfx allows to call the
resource update functions at any point in a frame, most importantly within render passes.

### The resource barrier system

This was by far the biggest hassle and took a long time to get right, involving
several rewrites (and there's *still* quite a lot of room for improvement).

The first implementation phase was basically to come up with a general barrier
insertion strategy which isn't completely dumb yet still satisfies the Vulkan
default validation layer, the second and much harder step was then to also satisify
the optional synchronization2 validation layer (which even most 'official'
Vulkan samples don't seem to get right - go figure).

I won't bore you with what Vulkan barriers are or why they are necessary, just
that barriers are usually needed when a Vulkan buffer or image changes the way
it is accessed by the CPU or GPU (for instance when a resource changes from
being a staging-upload target to being accessed by a shader, or when an image
object changes from being used as a pass attachment to being sampled as a
texture).

In sokol-gfx I tried as much as possible to use a 'lazy barrier system', e.g.
a barrier is inserted at the latest possible moment before a resource is used.

The basic idea is that sokol-gfx buffers and images keep track of their current
'access state', this may be a combination of:

- staging upload target
- vertex buffer binding
- index buffer binding
- read-only storage buffer binding
- read-write storage buffer binding
- texture binding
- storage image binding (always read-write)
- a pass attachment (in the flavours color, resolve, depth or stencil)
- a special 'discard' access modifier for pass attachments
  (used with `SG_LOADACTION_DONTCARE`)
- swapchain presentation

Implicity those access states carry additional information which may be needed
for picking the right barrier type, like whether shader accesses are read-only,
read-write or write-only, and whether the access may happen exclusively in
compute passes, render passes, or both.

Ideally barriers would always be inserted right at the point before a resource
is bound (because only at that point it's clear what the new access state is).

Unfortunately it's not that simple: there's a metric shitton of arbitrary
restrictions in Vulkan where exactly barriers may be inserted. The main
limitation is that no barriers can be inserted between `vkBeginRendering` and
`vkEndRendering` (which is hella weird, it would be obvious to disallow barriers
that involve the current pass attachments, but not for any other resources used
in the pass).

This limitation is currently the main reason why the sokol-gfx barrier system
is not optimal in some cases, because it requires to move any barriers that would
be inserted inside render passes before the start of the render pass. However sokol-gfx
can't predict what resources will actually be used in the render pass
(spoiler: there's a surprisingly simple solution to this problem which I
should have thought of myself much earlier - but that will be for a later
Vulkan backend update).

Currently, barriers insertion points are in the following sokol-gfx functions:

- `sg_begin_pass()`
- `sg_apply_bindings()`
- `sg_end_pass()`
- all staging functions

The obvious barriers in begin- and end-pass are for image objects transitioning
in and out of attachment state.

In `sg_apply_bindings()` barriers are only inserted inside compute passes (because
of the above mentioned 'no barriers inside render passes' rule).

In staging operations, barriers are issued at the start and end of the staging
operation, the 'after-barrier' is not optimal and eventually needs to be fixed.

Now the tricky part: moving barriers out of render passes... there is one
situation where this is relevant: a compute pass writes to a buffer or
image, and that buffer or image is then read by a shader in a render pass. Ideally
the barrier for this would happen inside the render pass in `sg_apply_bindings()`,
but Vulkan validation layer says "no".

What happens instead is that any resource that's (potentially) written in a
compute pass is tracked as 'dirty', and then in the `sg_end_pass()` of the compute
pass, very conservative barriers are inserted for all those dirty resources.
'Conservative' means that I cannot predict how the resource will be used next,
so buffers are generally transitioned into 'vertex+index+storage-buffer access
state' and images are generally transferred into 'texture access state'.

This generally appears to work but is not optimal. We'd like to delay those
barriers to when the resources are actually used, and also tighten the scope
of the barriers to their actual usage.

The solution for this is surprisingly simple: use the same 'time warp' that is
used for recording staging operations by recording barrier commands that would
need to be issued from within sokol-gfx render passes into a separate command
buffer which can then be enqueued **before** another command buffer which holds
all render/compute commands for the pass.

This is a perfect solution but requires a couple of changes which I didn't want
to do in the first Vulkan backend release to not push that out even further:

- instead of a single command buffer per frame to hold all render/compute
  commands, one command buffer per sokol-gfx pass is needed
- for render passes, a separate command buffer per pass is needed to record
  barrier commands so that the barriers can be moved out of Vulkan's
  `vkBeginRendering/vkEndRendering`

...inside `sg_apply_bindings()` and `sg_end_pass()` we're now doing some serious
time-travelling-shit:

Each resource that's used in a render pass will keep track of all the 'access
states' it's used as in the `sg_apply_bindings` call (for buffers that may be
vertex-, index- or read-only-storage-buffer-binding and for images it can only
be texture-binding), additionally the resource is uniquely-added to a tracking
array.

In `sg_end_pass()` we now have a list of all bound resources and their binding
types, and this information can be used to record 'just the right' barriers into
the **separate** command buffer that's been set aside for render pass
barriers. This barrier command buffer is then enqueued **before** the command
buffer which holds the render commands for that pass and voila: perfectly scoped
render pass barriers. But as I said, this will need to wait until a followup
update.

### Everything else...

The rest of the Vulkan backend is so straightforward that it's not
worth writing about, essentially 1:1 mappings from sokol-gfx API functions
to Vulkan API functions (the blog post is long enough as it is).

Apart from the resource update system (which is overly restrictive and
conservative in sokol-gfx, mainly because of OpenGL/WebGL), the sokol-gfx API
actually is a really good match for Vulkan. There are no expensive operations
(like creating and discarding Vulkan objects) happening in the 'hot-path'. The
use of `EXT_descriptor_buffer` is not a great choice for some GPU architectures,
but as I said at the start: I'm waiting for Khronos to finish their new resource
binding API which apparently will be a mix of D3D12-style descriptor heaps and
`EXT_descriptor_buffer`.

The next steps will most likely be:

- porting the backend to Windows (still limited to Intel GPU though)
- port the backend to NVIDIA (will have to wait until around January because
  I'll be away from my NVIDIA PC for the rest of the year)
- expose a GPU memory allocator interface, and add a sample which hooks up VMA
- ...maaaybe integrate SebAaltonen's OffsetAllocator as default allocator
  (still not clear if I need that when all modern Vulkan drivers no longer
  seem to have that infamouse 4096 unique allocations limit)
- tinker around with GPU memory heap types for uniform- and descriptor-buffers
  on GPUs without unified memory (e.g. host-visible + device-local)
- figure out why exactly RenderDoc doesn't work (apparently it's because
  of `EXT_descriptor_buffer`, but RenderDoc claims to support the extension since 1.41)
- add support for debug labels (not much point to implement this before
  RenderDoc works)
- implement the improved resource barrier system outlined above
- add support for multiple swapchain passes (not needed when used with sokol_app.h,
  but required for any 'multi-window-scenario')
- improve interoperability with Vulkan code that exists outside sokol-gfx
  (injecting Vulkan buffers and images into `sg_make_buffer/sg_make_image`
  and add the missing `sg_vk_query_*()` functions to expose internal
  Vulkan object handles)

Originally I also had a long rant about the Vulkan API design in this
blog post, maybe I'll put that into a separate post and also
change the style from rant into 'constructive criticism' (as hard as that will be lol).

My verdict about Vulkan so far is basically: Not great, not terrible.

It's better than OpenGL but not as good (from an API user's perspective) as pretty
much any other 3D API.  In many places Vulkan is already the same mess as
OpenGL. Sediment layers of outdated, deprecated or competing features and
extensions which is incredibly hard to make sense of when not closely following
Vulkan's development since its initial release in 2016 (which is the exact same
problem that ruined OpenGL).

At the very least, please, please, PLEASE aggressively remove cruft and reduce
the 'optional-features creep' in minor Vulkan API versions (which I think should
actually be major versions - 4 breaking versions in 10 years sounds just about right).

For instance when I'm working against the Vulkan 1.3 API I really don't care
about any legacy features which have been replaced by newer systems (like
synchronization2 replacing the old synchronization API). Don't expose the
extensions that have been incorporated into core up to 1.3, and also let me filter
out all those outdated declarations from the Vulkan headers so that Intellisense
doesn't suggest outdated declarations. Don't require me to explicitly enable any
little feature (like anisotropic filtering) when creating a Vulkan device. If
some shitty old-school GPU doesn't have anisotropic filtering, then just
silently ignore it instead of polluting the 3D API for all eternity just for
this one GPU model which probably wasn't even produced anymore even back in
2016.

Vulkan profiles are a good idea in theory, but please move them into the
core API instead of implementing them as a Vulkan SDK feature. Give
me a `vkCreateSystemDefaultDevice(VK_PROFILE_*)` function to get rid of those
500 lines of boilerplate that **every single Vulkan programmer** needs to
duplicate line by line anyway (people who need more control about the setup
process can still use that traditional initialization dance).

And PLEASE get somebody into Khronos who has the power to inject at least a
minimal amount of taste and elegance into Vulkan and who has a clear idea what should
and shouldn't go into the core API, because just promoting random vendor
extensions into core is really not a good way to build an API (and that was
clear since OpenGL - and the **one** thing that Vulkan should have done better).

Also, a low-level and efficient API **DOES NOT HAVE TO BE** a hassle to use.

Somehow modern software systems always seem be built around the 'no pain, no
gain' philosophy (see Rust, Vulkan, Wayland, ...), this sort of self-inflicted
suffering for the sake of purity is such a weird Christian flex that
I'm starting to wonder if 'religious memes' surviving under the surface in even
the most rational and atheist developer brains is actually a thing...

Maybe we should return to the 'Californian hippie attitude' for building computer
systems and software - apparently that had worked pretty great in the 70's and 80's ;)

...ok I'm getting into old-man-yells-at-cloud-mode again, so I'll better stop here :D
