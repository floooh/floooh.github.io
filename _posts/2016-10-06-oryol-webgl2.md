---
layout: post
title: WebGL2 is coming... 
---

...to the [Oryol](https://github.com/floooh/oryol) Gfx module. Or more
specifically: a number of new rendering features that bump the Oryol Gfx module
to GLES3.0/WebGL2 feature level across all rendering backends. 

Since WebGL(1) and GLES2 support won't be dropped anytime soon, using these
new features requires a runtime check whether the feature is available (using
the existing Gfx::QueryFeature() function).

Work is currently happening on the _webgl2_ branch, that's also the reason
why the master branch is so quiet at the moment (...and Elite Dangerous).

I hope that the stuff lands in the master branch at least a couple of weeks
before Firefox and Chrome flip the WebGL2 switch. When this happens, only the
GL and Metal rendering backends will have all new features enabled immediately,
but the new stuff will trickle into the D3D11 and D3D12 rendering backends soon after.

In browsers, the GL renderpath will first try to create a WebGL2 context, and
if this fails, fall back to WebGL1 and disable all features that require
WebGL2. This way the same 'exe' can work across older and newer browsers, as
long as the application implements fallback codepaths for WebGL1. On all other
GL platforms (with the exception of the RaspberryPi, which AFAIK is stuck on
GLES2) the new features will 'just be there'.

Here's the bullet-point list of features that are coming to all rendering backends:

- proper MRT (Multiple Render Target) support with a Metal-like RenderPass API,
  this will also be supported on GLES2/WebGL1 if the EXT\_draw\_buffers
  extension is present
- MSAA support for offscreen render targets
- support for volume- and array-textures
- rendering into single texture surfaces (mipmaps, cubemap-faces, volume- and array-slices)
- transform feedback / stream output
- support for 10/10/10/2-bit pixel- and vertex-formats 
- ETC2 texture support (most likely not on desktop browsers though, only mobile)
- support for vertex-id and instance-id in the vertex shader (plus I recently added
  point-size support in the vertex-shader, but that's not a WebGL2 feature)
- (low prio): proper integer support in shaders
- (low prio): occlusion query support

And some internal things in the GL rendering backend:

- uniform buffer and uniform block support
- some GL extensions are now a 'core feature', most notably: hardware instancing, floating point
  pixel formats (although rendering to floating point textures is still an extension AFAIK),

A closer look at the more interesting features:

### Uniform Buffer Support

From the outside, the Oryol Gfx module already uses uniform-block-like
C-structs for updating shader parameters, but what actually happens under the
hood depends on the rendering backend.

Oryol's shader-meta-language allows to group shader parameters into uniform blocks, which
are code-generated into C-structs plus some additional information about the structure layout
(names, offsets and types of struct members). Applying such a uniform block is a single
operation in all non-GL rendering backends (Metal, D3D11 and D3D12), but on the GL backend,
the update operation does one glUniform() call per struct member.

This has been fixed in the GL backend (if uniform buffers are available) to work more like
the Metal or D3D12 backend:

- On the shader side, uniform block structures will be generated in 'std140'
  layout ('std140' defines certain alignment and padding rules for uniform block members)
- A matching C-struct is created with the same alignment rules for each
  shader-uniform-block
- In the GL render backend, a double-buffered uniform buffer is created with
  enough room to hold all uniform updates for an entire frame
- On platforms with glMapBuffer() support (basically: everything but WebGL2),
  uniform block updates will be copied directly into the per-frame uniform
  buffer. On WebGL2, the copy goes into an intermediate memory buffer, followed
  by a single glBufferSubData() at the end of the frame to copy the uniform
  data over into the uniform buffer
- Since all uniform updates need to be finished before the first draw-call, all
  Gfx module rendering actions are now recorded into an internal command
  buffer, and played back in Gfx::CommitFrame(), thankfully this is very little
  additional code since the Gfx API is so simple. It is interesting that this
  makes the GL rendering backend very similar to the modern 3D APIs (Metal or
  D3D12). This new record/playback code may also open up the path towards exposing
  command buffers in the public Gfx interface.
- When the ApplyUniformBlock render command is played back, it will just do a
  glBindBufferRange() to notify the next draw call where its uniform data
  starts.

If the WebGL/GLES2 fallback path is used, everything will work as before. The
entire command-buffer recording is disabled, uniform updates happen through
granular glUniform() calls, and all GL calls happen immediately, not deferred
until Gfx::CommitFrame().

Unfortunately there is one pretty big downside with the uniform-buffer approach
in WebGL:

There's the dreaded GL\_UNIFORM\_BUFFER\_OFFSET\_ALIGNMENT which is 256 bytes
on the hardware configs I tested on (Intel GPU on OSX, and NVIDIA GPU via ANGLE
on Win10). This means the uniform block 'slots' in a uniform buffer must be at
least 256 bytes apart, even if an uniform update is only a single vec4 (16
bytes) or mat4 (64 bytes). On other 3D APIs this just wastes memory, but on
WebGL2 (which doesn't have glMapBuffer), it also means that a lot of empty
bytes need to be copied during the glBufferSubData() call which updates the
per-frame uniform buffer.

I am currently not seeing any speedups with the new approach in my draw-call
stress test. This is somewhat understandable because the test is only doing a
single vec4 glUniform update per draw-call, which is the worst case for the new
uniform-buffer code. To get some more real-world numbers I need to write a new
test with more complex uniform updates.

### Render Passes and Multiple Render Targets

So far, offscreen rendering was quite limited in Oryol. It was only possible to
render to a single color/depth-stencil surface pair without MSAA-support,
and reuse the color-render-target as texture later in the frame. 

The WebGL2 update will make offscreen rendering much more powerful, but
requires a small API change. The Gfx::ApplyRenderTarget() function will go away
and be replaced with a Gfx::BeginPass()/Gfx::EndPass() pair, and there will be
a new 'RenderPass' resource type. These changes bring the Gfx API more in line
with the modern 3D-APIs which also have the concept of render passes.

The new RenderPass object describes what should happen at the 
start and end of a pass, this looks all very similar to Metal:

There are 1..N color attachments, and an optional depth/stencil attachment, each
attachment is defined by:

- a texture object where rendering goes to (can be a 2D, 3D, array or 
  cubemap texture)
- mipmap / face / slice indices into the attachment texture
- default clear-values for color, depth and stencil (these can later be 
  overridden in Gfx::BeginPass())
- a 'load action' (what happens in BeginPass):
    - _DontCare_: the starting content of the attachment is undefined
    - _Clear_: the attachment will be cleared
    - _Load_: the attachment will keep its previous content
- a 'store action' (what happens in EndPass, I'm not sure about those yet,
  it may be that the Resolve will be implicit if the render-target 
  has been created as an MSAA texture):
    - _DontCare_: content of the attachment will be discarded
    - _Store_: content will be preserved
    - _Resolve_: MSAA-resolve into a second texture, throw-away content
      of MSAA render-surface
    - _StoreAndResolve_: perform MSAA-resolve, and preserve content
      of MSAA render-surface
    
There will be 4 versions of Gfx::BeginPass():

- **Gfx::BeginPass()**: this starts rendering into the 'default framebuffer' 
  using the default clear values defined upfront during Gfx::Setup()
- **Gfx::BeginPass(clearValues)**: start rendering into the default framebuffer,
  with override clear values
- **Gfx::BeginPass(renderPass)**: start an offscreen render-pass, use default clear-values
  defined during RenderPass creation
- **Gfx::BeginPass(renderPass, clearValues)**: start an offscreen render-pass, with
  override clear values

There's one big design-wart which will most likely change: at the moment the render
target textures are baked into the RenderPass at creation time, which may lead 
to a combinatorial explosion for stuff like ping-pong rendering (alternating between
reading from a texture and writing into another one). I'll see how it feels, and 
what restrictions are imposed by the various 3D APIs, I would prefer 
to set render target textures into slots when Gfx::BeginPass() is called,
similar to the mesh and texture slots in Gfx::ApplyDrawState().


### Transform Feedback

...AKA the poor man's Compute :) Since GL and D3D can't agree on a name (Transform Feedback
vs Stream Output) I invented a new name and call it herewith 'VertexCapture', which makes
a lot more sense anyway :P

This allows to capture the output of a vertex shader back into a vertex buffer
without CPU intervention. This is by far not as nice as real compute
capabilities, which are only in GLES3.1 and unfortunately didn't make it into
WebGL2, but it's still a nice alternative for moving CPU work onto the GPU.

Something similar was already possible before in WebGL1 by rendering to a
texture which could then be sampled in the vertex shader, but this often lead
to complex and/or ugly fragment shader code because the target buffer is just
a grid of pixels. With transform feedback it's possible to write elements that
look more like typical structs and which can directly feed back into the vertex
shader.

I think it's best to combine both methods (writing to textures from the
fragment shader which are then sampled in the vertex shader, *and* additionally
use transform feedback). The only important thing (especially in the browser)
is to get as much (trivially parallelizable) work from the CPU onto the GPU. 

As I said, true compute would be much, much better, especially for WebGL,
but beggers can't be choosers :)

The changes in the Gfx module API are minimal:

- when creating a Pipeline object, the PipelineSetup struct has 2 new members:
    - _EnableVertexCapture_: must be set to true to enable vertex capture
    - _CaptureLayout_: this is the vertex layout of the captured vertex shader output,
      this must match the output-capture definition in the vertex shader
- in the DrawState struct, there's a new slot _CaptureMesh_, if a mesh resource
  is set in this slot, it must have a vertex buffer where the capture result
  will be written to

And finally, the shader meta-language has a new syntax to mark vertex shader output
variables for capture. This new syntax associates a vertex shader output variable
with an input vertex attribute name, for instance:

```text
@vs
@in vec3 position
@in vec3 normal
@out vec3 pos => position
@out vec3 vel => normal
    vel = ...;
    pos = ...;
@end
```

Have a look at the @out definitions with the '=>', the line **@out vec3 pos => position** means: 

Capture the value of output variable 'pos' into the vertex attribute 'position' (...in the
capture vertex buffer).

## Sneak Preview Sample Webpage

I've uploaded the work-in-progress WebGL2 Oryol samples to a new location:

[https://floooh.github.io/oryol-webgl2/](https://floooh.github.io/oryol-webgl2/)

**Before clicking on that link**, note that these demos might crash your graphics
driver, especially on Windows with an Intel GPU.

Also, you'll need Firefox Nightly or Chrome Canary, and in Canary, WebGL2 must be enabled
via _chrome://flags/_

WebGL2 seems to be black-listed on many hardware configs at the moment. For
instance in Chrome Canary on a Windows7 machine with NVIDIA 760 GPU, the
samples silently fall back to WebGL1 because the WebGL2 context can't be
created. 

It's all very 'beta' at the moment.

To check whether the WebGL2 render path is actually used, search for the string
'WebGL 2.0' in the text field below the 3D canvas.

And that is all for today :)

