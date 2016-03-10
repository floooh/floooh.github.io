---
layout: post
title: Oryol Gfx Module Changes
---

I did some changes to Oyrol's Gfx module which slightly break existing code,
for the required changes it is best to look at the 
[Oryol sample source code](https://github.com/floooh/oryol/tree/master/code/Samples).

Without further ado:

### 'drawState' renamed to 'pipeline'

The 'draw-state' resource type has been renamed to 'pipeline', since it
maps now exactly to a D3D12 pipeline-state-object, and is a superset of similar
bundled render-state on other 3D APIs.

The only publicly visible class which is affected is the old DrawStateSetup
class, which is now called PipelineSetup.

### PipelineSetup takes VertexLayouts instead of Meshes

Previously it was necessary to provide one or more actual meshes when 
creating a draw state (from now on: pipeline), and a separate pipeline
object had to be created for every possible combination of input meshes.

Under the hood this wasn't as bad as it sounds, since all rendering backends
had some sort of caching which would re-use and share state-objects with the
same creation parameters, but it looked pretty dumb in some
realworld-situations. For instance, this new [voxel
demo](http://floooh.github.io/voxel-test/) allocates a pool of several hundred
dynamically updated vertex buffers, and previously, it was required
to create just as many pipeline objects (but because of the internal caching, 
only one D3D12 pipeline-state-object was actually created)

Now, a pipeline object only needs to know the vertex-layouts and the
primitive type of the input geometry, and it will be compatible with
any input mesh which matches the vertex-layouts provided at creation time.

Another nice side-effect of this change was the removal of the
complex state-object caching, eliminating a couple hundred lines of code.

### More flexible input mesh combinations

This is also a change inspired by the voxel demo: the demo uses a
single static buffer for indices, but hundreds of dynamically updated vertex
buffers, which was a combination that was simply not possible before without
creating many identical index buffers.

It is now possible to mix dynamic and static index- and vertex-buffers in 
the same mesh, and it is possible to create a mesh which has only an index
buffer. In this case, another mesh object must provide the vertex data. This
makes it possible to combine a single static index buffer with many different
dynamic buffers.

The obvious question is now: why have mesh objects at all, and not provide
simple vertex- and index-buffers to begin with? The reason is that this is
currently more convenient when loading asset data. You provide an URL of a
'mesh file', and get a single resource object back with all the data in the mesh
file under a single handle. This is an open topic though, if a simple solution
pops up, I'll likely move to separate vertex- and index-buffers.

### A new DrawState struct for passing resource ids to Gfx::ApplyDrawState()

The draw-state concept hasn't disappeared, just re-purposed: there is now a
public DrawState struct, which holds the handles for all resources that need to
be bound via Gfx::ApplyDrawState() for the following draw calls.

The DrawState structure basically represents the resource binding model of
Oryol in a simple and obvious way: resources must be plugged into 'resource
slots' before draw calls can be issued:

- there's one slot for a pipeline object, which defines the shader, render
  states and expected vertex layout
- slots for 1..4 meshes providing vertex and index data, the vertex
  layout of the meshes must match the expected vertex layout of the pipeline
  object
- slots for 0..4 textures for the vertex shader stage
- ...and slots for 0..12 textures for the fragment shader stage

The max number of meshes and textures can be tweaked through code constants in
the header Gfx/Core/GfxConfig.h, it might be a better idea to put these into
cmake options later.

### Shader code generator changes

Since textures are now plugged into the new DrawState struct, the shader code
generator no longer generates 'texture block' structures, instead it will
generate slot index constants for use with the texture bind slots in
the DrawState structure.

The @uniform and @texture tags have been removed, these were leftovers
from the time when uniform and texture blocks didn't exist.

Previously, a uniform block might have looked like this:

```cpp
@uniform_block vsParams VSParams
@uniform mat4 mvp ModelViewProjection
@uniform mat4 model Model
@uniform vec4[6] normal_table NormalTable
@uniform vec4[32] color_table ColorTable
@uniform vec3 light_dir LightDir
@uniform vec3 scale Scale
@uniform vec3 translate Translate
@uniform vec3 tex_translate TexTranslate
@end
```

Since uniforms *have* to be declared inside a uniform block, the @uniform tag
was redundant, so now it looks like this:

```cpp
@uniform_block vsParams VSParams
  mat4 mvp ModelViewProjection
  mat4 model Model
  vec4[6] normal_table NormalTable
  vec4[32] color_table ColorTable
  vec3 light_dir LightDir
  vec3 scale Scale
  vec3 translate Translate
  vec3 tex_translate TexTranslate
@end
```
The same idea applies to texture blocks.

### What's next

The next big thing for the Gfx module is Vulkan support, and with this
the Oryol Gfx module will very likely get a new render-pass resource
type, and (finally) support for multiple-render-target rendering.

