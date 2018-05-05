---
layout: post
title: Upcoming Oryol Changes (sokol_gfx.h migration)
---

I have mostly finished replacing the C++ rendering backends with sokol_gfx.h
in the Oryol Gfx module now, only things missing is testing and bugfixing
for Android and RaspberryPi. The changes are tracked in this
[pull request](https://github.com/floooh/oryol/pull/298).

Since the changes are fairly big anyway, I took the opportunity to
also cleanup a few things about module initialization and
Gfx resource creation that were rolling around in the back of 
my head for some time but never were important enough to
break existing code. Piggybacking those changes on the sokol_gfx.h
integration which would break compatibility anyway seemed like the
right thing to do.

I'm planning to merge this into master around late May / early June,
so still plenty of time to get familiar with the changes :)

## Overview of High Level API Changes

### Option-bag structs are now called XxxDesc instead of XxxSetup

This is a purely cosmetical change to bring the naming closer to standard 3D
APIs (D3D also calls them DESC structures, in Metal they are called
Descriptor objects, of course Vulkan had to mess everything up again and uses
CreateInfo for resource creation, while using Descriptor for resource
binding stuff, and D3D12 takes the crown by having both DESC structures for
resource creation, and Descriptors for resource binding). But anyway...

### Hardwired Vertex Attribute Names are Gone

The old Gfx module had hardwired vertex component names and semantics.
This restriction has been removed and vertex attributes can now
be named freely.

### Gfx Mesh replaced with Buffer

This is a pretty big conceptional change. Previously the Gfx module had a
Mesh resource type which contained everything needed to describe geometry
data: a vertex buffer, an optional index buffer, a vertex
layout, an index type (none, 16- or 32-bit), and 'primitive groups' (submesh ranges).

This concept made runtime combinations of different geometry data sources
a bit awkward (e.g. combining static geometry with dynamically created
data, or sharing the same index buffer with many different vertex buffers),
but it simplified creation or loading of mesh data.

When writing sokol_gfx.h I experimented with dumping this high-level
resource type, and instead use generic buffer resources in
exactly the same way the lower-level 3D-APIs do. I was a bit
concerned about loss of convenience (creating an entire mesh resource
with a single call versus creating separate vertex- and index-buffers),
but in reality this wasn't so bad, and scenarios where different
geometry data sources are combined are much clearer and straightforward
now.

### Method Chaining for Desc structure setup

This is by far the most visible change, and also the one 
that went through the most rewrites until it felt 'right': Desc structures
are now initialized with method chaining. The main motivation for
this was that I realized how awkward writing code for Oryol
suddenly felt compared to C99's designated initialization for structs
used in sokol_gfx.h samples.

Here's some code:

This is how a render-target texture was setup in 'old' Oryol:

```cpp
auto rtSetup = TextureSetup::RenderTarget2D(128, 128, PixelFormat::RGBA8);
rtSetup.Sampler.WrapU = TextureWrapMode::Repeat;
rtSetup.Sampler.WrapV = TextureWrapMode::Repeat;
rtSetup.Sampler.MagFilter = TextureFilterMode::Linear;
rtSetup.Sampler.MinFilter = TextureFilterMode::Linear;
rtSetup.SampleCount = 4;
Id tex = Gfx::CreateResource(rtSetup);
```

This is how it looks like in C99 with sokol_gfx.h:

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .render_target = true,
    .width = 128,
    .height = 128,
    .format = SG_PIXELFORMAT_RGBA8,
    .wrap_u = SG_FILTER_LINEAR,
    .wrap_v = SG_FILTER_LINEAR,
    .min_filter = SG_FILTER_LINEAR,
    .mag_filter = SG_FILTER_LINEAR,
    .sample_count = 4;
});
```
Note how the desc-structure initialization can be injected
into the creation-call, it's basically optional named function arguments.

This is how it looks in 'new' Oryol:

```cpp
Id tex = Gfx::CreateTexture(TextureDesc()
    .RenderTarget(true)
    .Width(128)
    .Height(128)
    .Format(PixelFormat::RGBA8)
    .WrapU(TextureWrapMode::Repeat)
    .WrapV(TextureWrapMode::Repeat)
    .MinFilter(TextureFilterMode::Linear)
    .MagFilter(TextureFilterMode::Linear)
    .SampleCount(4));
```

The Gfx::CreateTexture() method takes a TextureDesc object
which can be initialized right in the call now. Non-default
options are set through chained setter methods. All in all
it looks quite similar to the designated init from C99, or
optional named arguments in other languages.

The method chaining can even be a bit more convenient than C99.
Let's say you want to create two textures which only differ
in their pixel format. You can create a common Desc structure
and modify the pixel format on-the-fly when creating the 
textures:

```cpp
// a TextureDesc struct with common parameters:
auto texDesc = TextureDesc()
    .Width(128)
    .Height(128)
    .WrapU(TextureWrapMode::Repeat)
    .WrapV(TextureWrapMode::Repeat);
// create a texture with RGBA8 format:
Id tex1 = Gfx::CreateTexture(texDesc.Format(PixelFormat::RGBA8));
// and another texture with RGBA4 format:
Id tex2 = Gfx::CreateTexture(texDesc.Format(PixelFormat::RGBA4));
```

What I like about the new 'inline' way to create resources is that
it follows the typical 'stream of thought', and it works nicely
with Intellisense autocompletion. 

The 'stream of though' for creating a texture is basically:

"I need a texture":
```cpp
Id tex = ...
```

"Textures are created with Gfx::CreateTexture()" (that's the only thing
one really needs to remember)
```cpp
Id tex = Gfx::CreateTexture(...
```

From here on, Intellisense kicks in and one only needs to follow the bread crumbs..., e.g. Gfx::CreateTexture() takes a TextureDesc argument:

```cpp
Id tex = Gfx::CreateTexture(TextureDesc()...
```

...and TextureDesc has a number of setter methods, also known to Intellisense:

```cpp
Id tex = Gfx::CreateTexture(TextureDesc().Width(...));
```

And that's it! IMHO the new API has a bit less cognitive burden than
before for the resource creation code flow.


## A Tour of the new Gfx API

The Gfx module API is now a bit more explicit. For instance
instead of having a single _CreateResource()_ overloaded method for all
resource types, there are now explicit _CreateBuffer()_, _CreateTexture()_, ... methods.

But let's start at the top:

```cpp
class Gfx {
public:
    /// setup Gfx module
    static void Setup(const GfxDesc& desc);
    /// discard Gfx module
    static void Discard();
    /// check if Gfx module is setup
    static bool IsValid();
```

Gfx module initialization and shutdown isn't much different than
before, except that the method chaining for GfxDesc now allows to
put the whole GfxDesc setup into the method call:

```cpp
Gfx::Setup(GfxDesc().Width(800).Height(600).Title("Hello World!"));
```

On to resource management, the whole concept of resource labels
and batch-destruction by label is unchanged, the same for 
looking up a shared resource by a Locator (which is basically
a resource sharing name):

```cpp
    /// generate new resource label and push on label stack
    static ResourceLabel PushResourceLabel();
    /// push explicit resource label on label stack
    static void PushResourceLabel(ResourceLabel label);
    /// pop resource label from label stack
    static ResourceLabel PopResourceLabel();
    /// destroy one or several resources by matching label
    static void DestroyResources(ResourceLabel label);
    /// lookup a resource Id by Locator
    static Id LookupResource(const Locator& locator);
```

As mentioned above, there is now an explicit creation function
for each resource type, each takes a Desc structure, and returns
a resource Id:

```cpp
    /// create a buffer object without associated data
    static Id CreateBuffer(const BufferDesc& desc);
    /// create a texture object without associated data
    static Id CreateTexture(const TextureDesc& desc);
    /// create a shader object
    static Id CreateShader(const ShaderDesc& desc);
    /// create a pipeline object
    static Id CreatePipeline(const PipelineDesc& desc);
    /// create a render-pass object
    static Id CreatePass(const PassDesc& desc);
```

The following functions are new and allow asynchronous setup
of buffers and textures. In the old module this was provided
through a user-derivable ResourceLoader classes, the new 
API is a bit more explicit, but much more flexible for 'edge 
scenarios':

```cpp
    /// allocate a buffer resource id (async resource creation)
    static Id AllocBuffer(const Locator& loc);
    /// initialize a buffer (async resource creation)
    static void InitBuffer(const Id& id, const BufferDesc& desc);
    /// set allocated buffer to failed resource state (async resource creation)
    static void FailBuffer(const Id& id);
    /// allocate a texture resource id (async resource creation)
    static Id AllocTexture(const Locator& loc);
    /// initialize a texture (async resource creation)
    static void InitTexture(const Id& id, const TextureDesc& desc);
    /// set allocated texture to failed resource state (async resource creation)
    static void FailTexture(const Id& id);
```

The _Query_ function group is conceptionally unchanged from before,
with only some cruft removed:

```cpp
    /// test if an optional feature is supported
    static bool QueryFeature(GfxFeature::Code feat);
    /// get the supported shader language
    static ShaderLang::Code QueryShaderLang();
    /// query the resource state of a resource
    static ResourceState::Code QueryResourceState(const Id& id);
```

There are now only two _BeginPass()_ overloads instead of four:

```cpp
    /// begin rendering to default render pass with override clear values
    static void BeginPass(const PassAction& action=PassAction());
    /// begin offscreen rendering with override clear colors
    static void BeginPass(const Id& passId, const PassAction& action=PassAction());
    /// finish rendering to current pass
    static void EndPass();
```

The _PassAction_ object now also supports method chaining, for instance
clearing the background at the start of a pass can now look like this:

```cpp
Gfx::BeginPass(PassAction().Clear(1.0f, 0.0f, 0.0f, 1.0f));
```

No changes in the _Apply_ function group:

```cpp
    /// apply view port
    static void ApplyViewPort(int x, int y, int width, int height, bool originTopLeft=false);
    /// apply scissor rect (must also be enabled in Pipeline object)
    static void ApplyScissorRect(int x, int y, int width, int height, bool originTopLeft=false);
    /// apply draw state (Pipeline, Meshes and Textures)
    static void ApplyDrawState(const DrawState& drawState);
    /// apply a uniform block (call between ApplyDrawState and Draw)
    template<class T> static void ApplyUniformBlock(const T& ub);
```

The _UpdateVertices()_ and _UpdateIndices()_ functions have been merged
into _UpdateBuffer()_, and the _UpdateTexture()_ function uses a new
_ImageContent_ struct to describe the update operation:

```cpp
    /// update dynamic vertex or index data (complete replace)
    static void UpdateBuffer(const Id& id, const void* data, int numBytes);
    /// update dynamic texture image data (complete replace)
    static void UpdateTexture(const Id& id, const ImageContent& content);
```

The main _Draw()_ method no longer takes a PrimitiveGroup index (since
primitive groups were part of the removed Mesh resource type), instead
it now takes an explicit _baseElement_ and _numElements_ arg:

```cpp
    /// submit a draw call
    static void Draw(int baseElement, int numElements, int numInstances=1);
```

There's also an overload which takes a PrimitiveGroup object (which
is just the baseElement and numElements arg grouped into one object):

```cpp
    /// submit a draw call with baseElement and numElements taken from PrimitiveGroup
    static void Draw(const PrimitiveGroup& primGroup, int numInstances=1);
```

The remaining stuff is unchanged:

```cpp
    /// commit (and display) the current frame
    static void CommitFrame();
    /// reset the native 3D-API state-cache
    static void ResetStateCache();
```

## More Resource Creation Examples

Creating a vertex buffer for a triangle. Note that you don't
need to set a buffer type, the default type of BufferDesc() is
a vertex buffer:

```cpp
const float vertices[] = {
    // positions            // colors (RGBA)
     0.0f,  0.5f, 0.5f,     1.0f, 0.0f, 0.0f, 1.0f,
     0.5f, -0.5f, 0.5f,     0.0f, 1.0f, 0.0f , 1.0f,
    -0.5f, -0.5f, 0.5f,     0.0f, 0.0f, 1.0f, 1.0f,
};
Id vbuf = Gfx::CreateBuffer(BufferDesc()
    .Size(sizeof(vertices))
    .Content(vertices));
```

Creating an index buffer for a quad made of 2 triangles:

```cpp
const uint16_t indices[2 * 3] = {
    0, 1, 2,    // first triangle
    0, 2, 3,    // second triangle
};
Id ibuf = Gfx::CreateBuffer(BufferDesc()
    .Type(BufferType::IndexBuffer)
    .Size(sizeof(indices))
    .Content(indices));
```

Creating an empty vertex buffer to be updated with dynamic
data each frame:

```cpp
Id vbuf = Gfx::CreateBuffer(BufferDesc()
    .Size(1024)
    .Usage(Usage::Stream));
```

Creating a multi-sampled multiple-render-target pass-object with 3 color- and 1 depth-image-attachment:

```cpp
auto rtDesc = TextureDesc()
    .Type(TextureType::Texture2D)
    .RenderTarget(true)
    .Width(OffscreenWidth)
    .Height(OffscreenHeight)
    .Format(PixelFormat::RGBA8)
    .MinFilter(TextureFilterMode::Linear)
    .MagFilter(TextureFilterMode::Linear)
    .SampleCount(4);
Id rtColor0 = Gfx::CreateTexture(rtDesc);
Id rtColor1 = Gfx::CreateTexture(rtDesc);
Id rtColor2 = Gfx::CreateTexture(rtDesc);
Id rtDepth = Gfx::CreateTexture(TextureDesc(rtDesc).Format(PixelFormat::DEPTHSTENCIL));

this->mrtPass = Gfx::CreatePass(PassDesc()
    .ColorAttachment(0, rtColor0)
    .ColorAttachment(1, rtColor1)
    .ColorAttachment(2, rtColor2)
    .DepthStencilAttachment(rtDepth));
```

Creating a Pipeline object for rendering 3D shapes with float3 vertex
positions and float4 vertex colors, with 16-bit indices and triangle strips,
the shader being generated by the Oryol shader-code-generator:

```cpp
Id pip = Gfx::CreatePipeline(PipelineDesc()
    .Shader(Gfx::CreateShader(Shader::Desc()))
    .PrimitiveType(PrimitiveType::TriangleStrip)
    .IndexType(IndexType::UInt16)
    .DepthWriteEnabled(true)
    .DepthCmpFunc(CompareFunc::LessEqual)
    .Layout(0, {
        { "position", VertexFormat::Float3 },
        { "color0", VertexFormat::Float4 }
    }));
```

## Asset Loading and Creation Helpers

All asset creation and loading tasks have been moved out of the Gfx
module into the Asset module, and the APIs have been changed to follow
the same method chaining philosophy as resource creation in the Gfx module
(although method chaining was used before already in the Asset module
classes, so it feels a lot like before).

Again, best demonstrated with sample code:

### ShapeBuilder

Instead of a single MeshSetup structure, the ShapeBuilder class
now returns a ShapeBuilder::Result structure, which contains
several embedded structures for creating resources. This change
was necessary because the Mesh resource type has been replaced
with the lower level Buffer resource type.

Here's some code to create everything necessary to render
different 3D shapes in separate draw calls:

```cpp
ShapeBuilder::Result shapes = ShapeBuilder()
    .RandomColors(true)
    .Positions("position", VertexFormat::Float3)
    .Colors("color0", VertexFormat::UByte4N)
    .Box(1.0f, 1.0f, 1.0f, 4)
    .Sphere(0.75f, 36, 20)
    .Cylinder(0.5f, 1.5f, 36, 10)
    .Torus(0.3f, 0.5f, 20, 36)
    .Plane(1.5f, 1.5f, 10)
    .Build();
drawState.VertexBuffers[0] = Gfx::CreateBuffer(shapes.VertexBufferDesc);
drawState.IndexBuffer = Gfx::CreateBuffer(shapes.IndexBufferDesc);
drawState.Pipeline = Gfx::CreatePipeline(PipelineDesc(shapes.PipelineDesc)
    .Shader(Gfx::CreateShader(Shader::Desc()))
    .DepthWriteEnabled(true)
    .DepthCmpFunc(CompareFunc::LessEqual)
    .SampleCount(4));
```

Note how the PipelineDesc() structure uses the shapes.PipelineDesc
object as a 'blueprint' and adds additional creation parameters.

The ShapeBuilder returns an array of PrimitiveGroups
in the Result object, this is needed later for 
rendering primitive ranges:

```cpp
Gfx::BeginPass();
Gfx::ApplyDrawState(this->drawState);
static const glm::vec3 positions[] = {
    glm::vec3(-1.0, 1.0f, -6.0f),
    glm::vec3(1.0f, 1.0f, -6.0f),
    glm::vec3(-2.0f, -1.0f, -6.0f),
    glm::vec3(+2.0f, -1.0f, -6.0f),
    glm::vec3(0.0f, -1.0f, -6.0f)
};
int primGroupIndex = 0;
for (const auto& pos : positions) {
    this->params.mvp = this->computeMVP(pos);
    Gfx::ApplyUniformBlock(this->params);
    Gfx::Draw(shapes.PrimitiveGroups[primGroupIndex++]);
}
Gfx::EndPass();
Gfx::CommitFrame();
```

### TextureLoader

The TextureLoader class completely wraps loading a texture
asynchronously for common texture formats (.dds, .ktx, .pvr).
The TextureLoader::Load() function takes a 'blueprint' TextureDesc
structure with creation parameters for the loaded texture:

```cpp
Id tex = TextureLoader::Load(TextureDesc()
    .Locator(texPath)
    .MinFilter(TextureFilterMode::LinearMipmapLinear)
    .MagFilter(TextureFilterMode::Linear)
    .WrapU(TextureWrapMode::ClampToEdge)
    .WrapV(TextureWrapMode::ClampToEdge));
```

That's all, the returned texture Id can be used for rendering right
away, but the actual rendering operations will be dropped until the
texture has finished loading.

Alternatively you can query the current resource state to find out
if texture loading has finished:

```cpp
if (Gfx::QueryResourceState(tex) == ResourceState::Valid) {
    // texture has finished loading...
}
```

### FullscreenQuadBuilder

Previously creating a fullscreen quad was handled as a special case
in the Gfx module. This has now been moved out into a helper class.
Currently it's not as flexible as it could be (vertex components
and vertex attributes are hardwired for instance), in the future
there will probably be some more control over the creation details.

Creating a Buffer and Pipeline object to render a fullscreen quad now looks
like this:

```cpp
auto fsq = FullscreenQuadBuilder().Build();
Id vbuf = Gfx::CreateBuffer(fsq.VertexBufferDesc);
Id pip = Gfx::CreatePipeline(PipelineDesc(fsq.PipelineDesc)
    .Shader(Gfx::CreateShader(Shader::Desc())));
```


### Mesh Data Loading

Oryol itself doesn't have support for loading mesh data from files,
since pure mesh data file formats aren't as standardized as texture
file formats. The [Oryol Extension Samples](https://github.com/floooh/oryol-samples) repository will contain
example code of how to load mesh and material data, as an example,
here's the asynchronous mesh loading code for the [Dragons sample](http://floooh.github.io/oryol-samples/asmjs/Dragons.html):

```cpp
void
Dragons::loadModel(const Locator& loc) {
    // start loading the .orb file
    IO::Load(loc.Location(), [this](IO::LoadResult res) {
        if (OrbLoader::Load(res.Data, "model", this->orbModel)) {
            this->drawState.VertexBuffers[0] = this->orbModel.VertexBuffer;
            this->drawState.IndexBuffer = this->orbModel.IndexBuffer;
            this->vsParams.vtx_mag = this->orbModel.VertexMagnitude;
            this->drawState.Pipeline = Gfx::CreatePipeline(PipelineDesc()
                .Shader(this->shader)
                .Layout(0, this->orbModel.Layout)
                .Layout(1, this->instanceBufferLayout)
                .IndexType(this->orbModel.IndexType)
                .DepthWriteEnabled(true)
                .DepthCmpFunc(CompareFunc::LessEqual)
                .CullFaceEnabled(true)
                .SampleCount(this->gfxDesc.SampleCount())
                .ColorFormat(this->gfxDesc.ColorFormat())
                .DepthFormat(this->gfxDesc.DepthFormat()));
            this->initInstances();
        }
    },
    [](const URL& url, IOStatus::Code ioStatus) {
        // loading failed, just display an error message and carry on
        Log::Error("Failed to load file '%s' with '%s'\n", url.AsCStr(), IOStatus::ToString(ioStatus));
    });
}
```

This looks quite similar than before, except for creating
separate vertex- and index-buffers, and the method chaining
when creating the pipeline object.

## A complete Triangle Sample

...I think that cover all important changes. To bring everything
together, here's what the new Triangle sample looks like (minus
the shader code but that hasn't changed):

```cpp
#include "Pre.h"
#include "Core/Main.h"
#include "Gfx/Gfx.h"
#include "shaders.h"

using namespace Oryol;

class TriangleApp : public App {
public:
    AppState::Code OnRunning();
    AppState::Code OnInit();
    AppState::Code OnCleanup();

    DrawState drawState;
};
OryolMain(TriangleApp);

AppState::Code
TriangleApp::OnInit() {
    // setup rendering system
    Gfx::Setup(GfxDesc().Width(400).Height(400).Title("Oryol Triangle Sample"));
    
    // create a mesh with vertex data from memory
    const float vertices[] = {
        // positions            // colors (RGBA)
         0.0f,  0.5f, 0.5f,     1.0f, 0.0f, 0.0f, 1.0f,
         0.5f, -0.5f, 0.5f,     0.0f, 1.0f, 0.0f , 1.0f,
        -0.5f, -0.5f, 0.5f,     0.0f, 0.0f, 1.0f, 1.0f,
    };
    this->drawState.VertexBuffers[0] = Gfx::CreateBuffer(BufferDesc()
        .Size(sizeof(vertices))
        .Content(vertices));

    // create shader and pipeline-state-object
    this->drawState.Pipeline = Gfx::CreatePipeline(PipelineDesc()
        .Shader(Gfx::CreateShader(Shader::Desc()))
        .Layout(0, {
            { "position", VertexFormat::Float3 },
            { "color0", VertexFormat::Float4 }
        }));

    return App::OnInit();
}

AppState::Code
TriangleApp::OnRunning() {
    
    Gfx::BeginPass();
    Gfx::ApplyDrawState(this->drawState);
    Gfx::Draw(0, 3);
    Gfx::EndPass();
    Gfx::CommitFrame();
    
    // continue running or quit?
    return Gfx::QuitRequested() ? AppState::Cleanup : AppState::Running;
}

AppState::Code
TriangleApp::OnCleanup() {
    Gfx::Discard();
    return App::OnCleanup();
}
```
