---
layout: post
title: "Thoughts about a WebGL-Next"
---
**TL;DR**: what a WebGL2 successor could look like and why it can't be 'WebVulkan'

The topic of 'WebVulkan' as a WebGL2 successor is coming up from time to time,
here's my 2ct since I also spend a lot of time thinking about what a
modern 3D-API 'for the rest of us' should look like.

For the record: I'm not associated with any browser, GPU or 3D-API maker, 
so this is purely from the 'receiving end' perspective ;)

**Here are the current drawbacks of WebGL** which should be fixed
in a hypothetical 'WebGL-Next' (all IMHO):

### The WebGL API is too granular

Just like GL, the WebGL API is too granular, too many calls are required per
frame to get something interesting on screen. In a native GL implementation
this isn't a big problem, it only becomes a problem when running in JS. GL
calls are cheap in a native environment, but much more expensive in a JS
engine, especially if JS objects are involved that might put pressure on the
garbage collector.

In WebGL1, the main problem areas are: render state updates, vertex attribute
definition, uniform updates. WebGL2 has vertex array objects and uniform blocks
which improve 2 of those. The granular render state update calls
remain a problem though.

Because of the unfortunate combination of a granular API and a high
call-overhead the same batching tricks like back in the D3D9 days must
be performed when using WebGL. A lot of code complexity is needed to avoid
redundant calls into WebGL, and even then the overhead is much higher compared
to talking to a good native GL driver.

The overhead also exists because WebGL needs to peform some additional
security-related checks (e.g. validate that vertex indexes are not
out-of-bounds, which makes dynamic buffer updates more expensive). Also, on
Windows, WebGL usually runs on top D3D9 or D3D11 instead of GL, which may
add additional overhead.

This results in very different performance characteristics between WebGL and
native GL implementations, code that performs well on OpenGL or GLES2 may
perform badly on WebGL. On the other hand, WebGL is secure and extremely well
tested for GLES2 conformance. The only 3D-API which comes even close to 'write
once run everywhere'.

From my experience, WebGL on desktop browsers can do about 5000 traditional
draw calls at 60Hz (1 uniform update + 1 draw call), the differences between
browser, underlying GPU or driver don't matter that much. WebGL code will be
much sooner CPU-bound than GPU bound, even on low-end GPUs, it is important to
keep that in mind when writing WebGL code.

For comparison, a good OpenGL driver on Windows reaches over 100k draw calls
before dropping below 60Hz (but at the same time a bad GL driver on an
integrated GPU only reaches slightly above 10k).

### WebGL can't directly access GPU memory

This is probably the most critical limitation for a move to a more modern 3D
API, but it exists for a good reason: Javascript is not allowed to directly
write to GPU-accessible memory because: security.

I am not *that* much into browser security tech to know whether this limitation
is forever cast in stone or whether there are watertight secure ways to 
write GPU memory directly from JS, but I think that would be a very tough
nut to crack for a 'WebGL next'. On the other hand, NaCl also did the 
'impossible' to run actual native code securely in the browser, so may be there's
a clever way around.

The problem is that the modern 3D APIs are pretty much all about directly
sharing memory between the CPU and GPU, and they don't give a shit about whether the
user-code does something wrong, instead the GPU will simply go belly up and
sometimes take the whole system with it. Which is not really acceptable on the
web of course.

### WebGL can't be efficiently threaded

There have been several attempts to drive WebGL from worker threads, but just
as the failed attempts in traditional native 3D APIs, not much came of it. The
modern 3D APIs have arrived at command lists to solve the threading problem in
a simple and straight-forward way. Instead of directly feeding the GPU from
different threads, command-lists are built on CPU-threads completely
independent from each other, and then moved over to the main thread where they
are queued for execution on the GPU.

### WebGL has no shader byte code

This is a relatively small problem compared to the others, but each browser
vendor has to implement its own GLSL compiler in the browser. And while
WebGL is very well conformance-tested, sometimes a shader compiles
fine in one browser but not in another. 

### Why a 'WebVulkan' makes no sense

Again all IMHO:

- obviously: making GPU memory directly accessible from JS would be a tough
  challenge to harden for security, but Vulkan is all about giving the CPU
  direct access to GPU resources
- Vulkan is not so much a 3D-API as a meta-API to build your own 3D-API. It no
  longer even pretends to be usable without a wrapper consisting of thousands
  of lines of boilerplate code. In that sense it is more an 'anti-API'. I haven't
  made up my mind yet whether this is good or bad. What I know is that writing
  Vulkan code feels like 'work', there's not a lot of fun in it. It just gives
  you a box of screws and bolts and says: here's everything you need to build
  your own flavour of D3D or GL. Go ahead and try to do better. D3D12 is nearly
  the same. **It is wasted effort to bring this mindset to the web**,
  especially since the performance advantages would be lost in the noise (even
  on native platforms, programmers struggle to beat D3D11 with their own
  specialized rendering code, writing Vulkan or D3D12 code that performs well
  across all GPU vendors is a really damn hard problem. Only Metal tries to
  offer a 3D API that's usable by mere humans without a sanity layer 
  inbetween.
- None of the modern 3D APIs are available on all platforms:
    - Vulkan is Windows/Linux/Android only
    - D3D12 is Windows10/UWP only
    - Metal is iOS/OSX only
- Trying to emulate Vulkan on D3D12 or even Metal (similar to what 
  ANGLE does with GLES2-on-D3D9) is a wasted effort. All the performance
  advantages would be lost, in exchange for an over-complicated API.

### Why a modern Web-3D-API still makes sense

Aside from the hideously complex manual resource management in Vulkan and D3D12
the new APIs also have good parts that would help to fix WebGL's shortcomings,
and even make it a simpler 3D-API than it is today:

- **Pipeline State Objects**: these are opaque objects which bundle all
  granular render state, vertex layout definition and shaders into a single,
  immutable object. This would easily be the one change with the most
  benefit for WebGL. Instead of dozens of granular calls, a single call would
  reconfigure the entire rendering pipeline. The user-code would no longer need
  to implement state caching in order to reduce calls into WebGL, and a whole
  class of bugs would simply vanish where the render pipeline
  configuration is left in a inconsistent state because of a few forgotten 
  render state updates.

- **Command Lists**: These would solve the whole 'rendering from web workers'
  in a very simple and elegant way. The worker thread only needs to know opaque
  resource handles (should it also be able to create resources? hmm...).
  Render commands are recorded into command lists on worker threads, and than moved
  over to the main thread where the command lists are enqueued for execution on
  the GPU.

- **Render Passes**: Vulkan- or Metal-style render passes enable an important
  class of optimizations for tiled-renderers, where multiple render passes can
  happen per-tile on the GPU without having to store pass-results in video
  memory.

- **SPIR-V shader byte code**: SPIR-V is easier to evaluate and cross-translate
  than GLSL. Shader language frontends would still make sense, these could be
  implemented as JS modules instead of baking them into the browser, but the
  'web guys' are now also getting used to a separate compile step (e.g.
  Typescript), so I don't see a problem with compiling shader code before
  deployment.

### Resource Management (buffers & images)

Here I would vote for ease-of-use instead of explicit control, the same way that
D3D11 and especially Metal does it.

Instead of explicitely managing all resource memory and resource state
transitions, the application code provides an intent how it is going to use a
resource, and the details are handled inside the 3D API just like in D3D11 or
(with slightly more control) in Metal.

It would be nice however to still get rid of redundant memory copies or granular
API calls. For instance  there *should* be a 'buffer mapping' where the JS code
can directly write vertex/index/uniform updates, and then may be call a
'didModifyRange' similar to Metal on OSX to inform the 3D API about dirty
regions. Under the hood, the browser would still need to validate the modified
data and very likely do a separate copy to GPU memory though.

It should be possible to record all dynamic shader uniform updates for an
entire frame into one big buffer without having to explicitely request access
to the buffer more than once (at the start of the frame), for each draw call,
record a simple buffer offset to tell the next draw call where its uniform data
starts, and only do a single 'didModifyRange' for the entire buffer right
before enqueueing the command list. This is possible in Metal, but not in D3D11
until D3D11.3 (if I remember right).

There's one important difference between D3D11 and Metal regarding dynamic
buffer updates: D3D11 has internal buffer renaming to make sure that
the CPU doesn't overwrite areas that the GPU might currently read. In Metal
the application code needs to take care of this. The simplest solution
in Metal is to implement a per-frame double-buffering scheme, and Metal
is open enough to allow more elaborate schemes.

## The End

So that's it. My highly inofficial proposal for the next WebGL ;) The big
difference to the 'WebVulkan' approach is that it focuses on a very 
simple API which would provide room for performance improvements 
over WebGL without the brute-force approach of D3D12 or Vulkan,
and it could even be implemented on top of GLES2 if needed.

The days of the 'One 3D-API To Rule Them All' are over, arguably this was never
true since OpenGL implementations differed so much in important details that
each GL implementation was its own little platform with different feature sets and
performance behaviour. WebGL is a bit of a 'unicorn' because it did the right
thing at the right time, and in a very pragmatic way.

Being very similar to GL/GLES2 definitely drove WebGL's success. But Vulkan
and D3D12 have a very different purpose than GLES2. They are not meant
for consumption 'by the rest of us', but written for a small elite of
highly specialized rendering engineers. In my opinion this is a dangerous
road never taken by any (surviving) 3D-API. Up until D3D11, each
D3D API became easier to use, performance improvements were a nice
side effect. In this regard, OpenGL was always a lost cause because
it simply packed new stuff on old stuff, resulting in a terrible
mess. 

When I look at the current situation, I feel thrown back to around 1997,
when each GPU vendor created its own shitty little 3D API, and Microsoft
created that terrible mess that was Direct3D3 (when I wrote the D3D12
backend for Oryol, long buried painful memories of D3D3's execute buffers
surfaced more than once).

The whole situation would be different if Metal would be a child of Khronos
with a plain C-API and supported by all platform owners. In that case the
situation would be crystal clear: let's create a WebMetal that maps 1:1 to this
hypothetical 'Khronos Metal', and may be in some alternate reality this is even
happening. But alas we're stuck in this shitty universe and need to come up
with another solution ;)

PS: if you want to get a better idea of what such a highly simplified 3D-API
could look like, have a look here at the [Oryol Gfx
interface](https://github.com/floooh/oryol/blob/master/code/Modules/Gfx/Gfx.h).
Granted, this leaves out a lot of modern stuff since it needs to map back to
GLES2/WebGL and it needs to work across all current 3D APIs, so it is a bit too
radical on the 'simplicity side' and too thin on the 'feature side', but it
demonstrates the basic idea of a simple-yet-modern 3D API well I think.


