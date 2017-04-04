---
layout: post
title: "Oryol: WebGL2 / RenderPass merge incoming"
---

If everything goes well I will merge my WebGL2/RenderPass work in Oryol back
to master next weekend (8/9-Apr-2017).

This post is about:

- changes which break existing code
- overview of new features
- overview of things that didn't make the cut
- and a quick outlook what's next

You can track the changes in full detail by looking through
this Pull Request:

[https://github.com/floooh/oryol/pull/247/files](https://github.com/floooh/oryol/pull/247/files)

### What's New

The following things are now possible:

- GLES3 is now used on iOS if the GL rendering backend is used
- WebGL2 is supported, but must be 'opted in' by using one of the ```webgl2-emsc-*-*``` build configs (e.g. ```./fips set config webgl2-emsc-make-debug```)
- if WebGL2 is not supported by a browser, the rendering backend will automatically fall back to WebGL
- 2D-array- and 3D-textures are now supported
- multiple-render-target offscreen-rendering is now supported
- rendering to cubemap-, 2D-array- and 3D-texture-slices is now possible
- MSAA offscreen rendering is now supported, with an automatic MSAA-resolve in Gfx::EndPass()
- a new 10.10.10.2 bit vertex format (VertexFormat::UInt10_2N) has been added
- ...and a new 10.10.10.2 bit renderable pixel format (PixelFormat::R10G10B10A2)

### What didn't make it

- I had originally added transform feedback support (capturing the output of
a vertex shader into a vertex buffer), I have removed this because
shader-code-generator support in Metal and D3D11 was too tricky, and I want
to fix the shader generation first before tackling this again (I would
prefer proper compute shader support in a WebGL2.1 of course)
- same with the uniform buffer support I had implemented in the new GL
renderer, I didn't see any performance difference to updating granular
uniforms on any platform I tested, but the code was much more complex (since
I was adding command recording- and playback to the GL renderer), I scrapped
the whole code for these 2 reasons (no performance difference, but much more
code) (note that the D3D11 and Metal renderer both use uniform buffers very
efficiently, only the GL renderer doesn't use them)
- ...and I also scrapped the experimental D3D12 renderer, I thought long and
hard about this for a couple of weeks, but I couldn't justify to keep it in
the end. The D3D12 renderer didn't show any performance difference to D3D11
(understandably, since the Oryol renderer is single-threaded), but it added
several thousand lines of non-trivial code

### What's coming up

I'd like to tackle the shader-code-generation next:

Cross-translating the 'meta-shader-code' to different GLSL versions, HLSL,
and MetalSL currently works through fragile text transformations, it becomes
harder and harder to add new features, and there are more and more 'magic
keywords' necessary when writing shader code. I want to replace this with
[glslang](https://github.com/KhronosGroup/glslang) to generate SPIR-V byte
code, and then translate this byte code to GLSL, HLSL and MetalSL with
[SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross).

At least that's currently my naive idea :)

### What needs to be changed

#### Gfx::ApplyRenderTarget() => Gfx::BeginPass()

The two _Gfx::ApplyRenderTarget()_ methods are gone:

```cpp
class Gfx {
    ...
    /// apply the default render target and perform clear-actions
    static void ApplyDefaultRenderTarget(const ClearState& clearState=ClearState());
    /// apply an offscreen render target and perform clear-actions
    static void ApplyRenderTarget(const Id& id, const ClearState& clearState=ClearState());
    ...
};
```
They have been replaced with 4 variants of _Gfx::BeginPass()_, and
a new _Gfx::EndPass()_ method:

```cpp
class Gfx {
    ...
    /// begin rendering to default render pass
    static void BeginPass();
    /// begin rendering to default render pass with override clear values
    static void BeginPass(const PassAction& action);
    /// begin offscreen rendering
    static void BeginPass(const Id& passId);
    /// begin offscreen rendering with override clear colors
    static void BeginPass(const Id& passId, const PassAction& action);
    /// finish rendering to current pass
    static void EndPass();
    ...
}
```

If your code is just rendering to the default framebuffer and 
doesn't change the default clear color, simply replace
the call to _Gfx::ApplyDefaultRenderTarget()_ with _Gfx::BeginPass()_
(without arguments), and add a _Gfx::EndPass()_ right before 
_Gfx::CommitFrame()_.

Before:

```cpp
Gfx::ApplyDefaultRenderTarget();
Gfx::ApplyDrawState(this->drawState);
Gfx::Draw();
Gfx::CommitFrame();
```

After:

```cpp
Gfx::BeginPass();
Gfx::ApplyDrawState(this->drawState);
Gfx::Draw();
Gfx::EndPass();
Gfx::CommitFrame();
```

If you render to the default framebuffer, but have a custom clear
color, you have 2 options now:

If the clear values are fixed, just define them once in the GfxSetup
struct by configuring the _DefaultPassAction_ when setting up the Gfx
module, and call _Gfx::BeginPass()_ without parameters:

```cpp
// setup Gfx module with custom clear color for the
// default framebuffer:
auto gfxSetup = GfxSetup::Window(800, 600, "Bla");
gfxSetup.DefaultPassAction = PassAction::Clear(glm::vec4(0.25f, 0.45f, 0.65f, 1.0f));
Gfx::Setup(gfxSetup);

// later in the render loop:
Gfx::BeginPass();
...
Gfx::EndPass();
Gfx::CommitFrame();
```

You can also define custom clear values for the default framebuffer by
calling _Gfx::BeginPass()_ with a _PassAction_ argument, this is useful
if you don't know the clear color yet at application start, or
if the clear color changes dynamically:

```cpp
Gfx::BeginPass(PassAction::Clear(glm::vec4(...)));
...
Gfx::EndPass();
Gfx::CommitFrame();
```

If you perform **offscreen rendering**, instead of _Gfx::ApplyRenderTarget()_ 
with a texture Id, you now need to call _Gfx::BeginPass()_ with the Id
of a render pass object.

Before:

```cpp
// create a render target texture:
auto rtSetup = TextureSetup::RenderTarget(128, 128);
rtSetup.ColorFormat = PixelFormat::RGBA8;
rtSetup.DepthFormat = PixelFormat::DEPTH;
this->rtTexture = Gfx::CreateResource(rtSetup);

// ...later in the render loop
Gfx::ApplyRenderTarget(this->renderTarget);
...
Gfx::CommitFrame();
```

This needs to be rewritten like this:

```cpp
// create a render target texture, and a render pass:
auto rtSetup = TextureSetup::RenderTarget2D(128, 128, PixelFormat::RGBA8, PixelFormat::DEPTH);
Id rtTexture = Gfx::CreateResource(rtSetup);
auto rpSetup = PassSetup::From(rtTexture, rtTexture);
this->renderPass = Gfx::CreateResource(rpSetup);

// ...later in the render loop
Gfx::BeginPass(this->renderPass);
...
Gfx::EndPass();
```

The default PassAction for offscreen rendering is to clear the
color buffer to black, the depth buffer to 1.0, and the stencil
buffer to 0.

Overriding the clear values follows the same idea as with the
default framebuffer. If the clear values are static, and don't
need to change, they can be predefined in the PassSetup structure when
the render pass is created:

```cpp
auto rpSetup = PassSetup::From(rtTexture, rtTexture);
rpSetup.DefaultAction = PassAction::Clear(glm::vec4(...));
this->renderPass = Gfx::CreateResource(rpSetup);
```

...or if the clear values are unknown yet when the pass object is
created, or they should change dynamically, hand a PassAction
object as second param to _Gfx::BeginPass()_:

```cpp
Gfx::BeginPass(this->renderPass, PassAction::Clear(glm::vec4(...)));
...
Gfx::EndPass();
```

#### TextureSetup initialization

The static creator functions for TextureSetup have been renamed, 
and new ones have been added. The new methods are:

```cpp
class TextureSetup {
    ...
    static TextureSetup FromPixelData2D(...);
    static TextureSetup FromPixelDataCube(...);
    static TextureSetup FromPixelData3D(...);
    static TextureSetup FromPixelDataArray(...);
    static TextureSetup Empty2D(...);
    static TextureSetup EmptyCube(...);
    static TextureSetup Empty3D(...);
    static TextureSetup EmptyArray(...);
    static TextureSetup RenderTarget2D(...);
    static TextureSetup RenderTargetCube(...);
    static TextureSetup RenderTarget3D(...);
    static TextureSetup RenderTargetArray(...)
    ...
};
```

Some other smaller changes:

* _Gfx::RenderTargetAttrs()_ has been renamed to _Gfx::PassAttrs()_
* _Gfx::ReadPixels()_ has been removed (this was only supported on GL anyay)

There may be more smaller changes (renamed public members), but these
should be straightforward to fix.

