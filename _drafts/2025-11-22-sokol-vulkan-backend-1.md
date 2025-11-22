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
  I haven't paid much attention to limitations of mobile GPUs
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
OpenGL backend (first, an alternative to WebGL2 had to happen - which has
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
