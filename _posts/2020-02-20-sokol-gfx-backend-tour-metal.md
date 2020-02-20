---
layout: post
title: "sokol_gfx.h Backend Tour: Metal"
---

This is part 3 of the blog post mini-series which looks under the hood of the
[sokol_gfx.h](https://github.com/floooh/sokol/blob/master/sokol_gfx.h)
backends. [Part
1](https://floooh.github.io/2020/02/17/sokol-gfx-backend-tour-gl.html) was
mainly about the OpenGL backend, [Part
2](https://floooh.github.io/2020/02/18/sokol-gfx-backend-tour-d3d11.html) about
the D3D11 backend, and this one is about the Metal backend.

The Metal backend is on one hand quite different from the other backends
(mainly because it is the only backend written in a different language:
Objective-C, instead of plain C), but it's also quite similar to the D3D11
backend (mainly because - at least if you squint a bit and ignore the language
differences between Objective-C and C - "version 1" of Metal looks a lot like a
carefully modernized and cleaned up version of D3D11, basically what D3D12
could have been if Mantle wouldn't have happend).

The sokol-gfx Metal backend has a few specialities not found in the other backends
which are worth looking at first:

## Objective-C, ARC and lifetimes

The Metal backend is written in Objective-C (not Objective-C++), and expects
ARC to be enabled (ARC = Automatic Reference Counting, which (as far as I
understand it) does static lifetime tracking for ObjC objects and inserts
refcounting calls where necessary). At the time I originally wrote the backend
this posed a little problem: ObjC object Ids couldn't be embedded into C
structs when compiled as Objective-C because the compiler couldn't track the
lifetime of such Ids inside C structs (this only worked when using
Objective-C++). In the meantime this feature seems to have been added, but I
think the solution I came up with is better anyway because it nicely separates
Objective-C "smart data" from C "dumb data".

The Metal backend stores all Objective-C object references in a single "Id
pool" (a simple NSMutableArray holding object references). Whenever sokol-gfx
creates an Objective-C object, it is stored in that array, and its array index
is returned. This index is then stored in C structs instead of the actual
object reference. Only when methods must be invoked on the Objective-C object,
the object is 'looked up' in the global 'reference pool' through the stored array
index.

This reference pool is also used to control the lifetime of all Objective-C
objects. As long as the object is registered in the pool it will be pinned into
memory from ARC's point of view, and in addition, this also allows to use
'unretained' Metal command buffers which avoid any reference counting overhead during
rendering (which can be quite significant).

When the Metal backend no longer needs an Objective-C object, it will be
'defer-released'.  This doesn't immediately destroy the object (since it might
still be 'in flight' for rendering), instead it will be added to a simple
'deferred-release queue' which will delete the objects a few frames later when
it is safe to do so.

## The "Tick Tock" Frame-sync Model

The entire Metal backend follows a simple 'tick-tock' frame-synchronization
model. While one set of resources is 'in flight' and used to render the current
frame on the GPU, another set of resources is updated with new data for the
next frame on the CPU side.  There is a single sync-point at the start of a
new frame where the CPU side needs to check whether (or worst case: wait
until) the frame before the current 'in-flight' frame has finished. The
tick-tock frame model also provides a simple way to decide when it is safe to
delete a resource, which is 3 frames into the future.

## The Sampler Cache

The Metal backend implements a very simple cache for reducing the number of
sampler-state objects that are created. This isn't strictly necessary, because
Metal doesn't restrict the total number of samplers, but since sampler state is
part of the image state in sokol-gfx, and a typical application will only have
a handful of different sampler states, it makes sense here to not create redundant
Metal sampler state objects.

## Per-frame Uniform Buffers

The last speciality of the Metal backend is how it handles shader uniform data:

When the Metal backend is initialized, two big uniform buffers (one per
'tick-tock frame') are created, big enough to hold all uniform updates for one
frame (this size is runtime-tweakable via the ```sg_desc``` struct handed to
```sg_setup()```, but must be known upfront, the default-size is 4 MBytes per buffer).

During the frame, each call to ```sg_apply_uniforms()``` copies the new
uniform-update-data into the current 'tick-tock' uniform buffer, records the
offset to the new data into the Metal command encoder, and advances the offset
for the next update taking alignment restrictions into account.

At the end of the frame, and only on macOS, the updated uniform data must 
be 'flushed' by calling a 'didMofifyRange' method on the current uniform buffer.

The same repeats in the next frame, but with the other 'tick-tock' uniform buffer.

That's it for all the noteworthy 'specialities' of the Metal backend.

## API Function Mapping

Finally, here's how the sokol-gfx backend functions map to Metal API calls (or
generally macOS system calls):

### _sg_mtl_setup_backend()

- the 'registry pool' for Objective-C object Ids is created and filled with null
references:
    - **[NSMutableArray arrayWithCapacity:]**
- a 'Grand Central Dispatch' semaphore object for syncing the CPU and GPU side is created:
    - **dispatch_semaphore_create()**
- a Metal command submission queue is created:
    - **[MTLDevice newCommandQueue]**
- the two per-frame uniform-buffers are created:
    - 2x **[MTLDevice newBufferWithLength:options:]**

### _sg_mtl_discard_backend()

- wait for 'in-flight' frames to finish before we start destroying stuff:
    - **dispatch_semaphore_wait()**
    - all stored Objective-C references are cleared, which in turn causes ARC
      to destroy the referenced objects

### _sg_mtl_reset_state_cache()
### _sg_mtl_create_context()
### _sg_mtl_destroy_context()
### _sg_mtl_activate_context()

- no Metal functions called in these backend functions

### _sg_mtl_create_buffer()

- **[MTLDevice newBufferWithBytes:length:options]** (if immutable buffer) OR
- 2x **[MTLDevice newBufferWithLength:options]** (if dynamic usage)

### _sg_mtl_destroy_buffer()

- 1x or 2x 'defer release' on the created buffers

### _sg_mtl_create_image()

- **[[MTLTextureDescriptor alloc] init]**
- 1x or 2x **[MTLDevice newTextureWithDescriptor:]**
- if immutable image, need to copy content:
    - for each face (1 or 6):
        - for each mip surface:
            - for each 3D- or array-slice:
                - **[MTLTexture replaceRegion:...]**
- if MSAA render target, create an additional MSAA texture: **[MTLDevice newTextureWithDescriptor:]**
- if new unique sampler state:
    - **[[MTLSamplerDescriptor alloc] init]**
    - **[MTLDevice newSamplerStateWithDescriptor:]**

### _sg_mtl_destroy_image()

- up to 3x 'defer release' on the created textures
- sampler-states are not released until sg_shutdown()

### _sg_mtl_create_shader()

- 2x (for vertex- and fragment-shader):
    - if shader byte code provided:
        - **dispatch_data_create()** (this is essentially an auto-released dynamic memory buffer)
        - **[MTLDevice newLibraryWithData:error:]**
    - otherwise if shader source code provided:
        - **[NSString stringWithUTF8String:]** (for source code)
        - **[MTLDevice newLibraryWithSource:options:error:]**
    - create function objects:
        - **[NSString stringWithUTF8String:]** (for entry function name)
        - **[MTLLibrary newFunctionWithName:]**

### _sg_mtl_destroy_shader()

- 2x 'defer release' for MTLLibrary
- 2x 'defer release' for MTLFunction

### _sg_mtl_make_pipeline()

- **[MTLVertexDescriptor vertexDescriptor]**
- **[[MTLRenderPipelineDescriptor alloc] init]**
- **[MTLDevice newRenderPipelineStateWithDescriptor:error:]**
- **[[MTLDepthStencilStateDescriptor alloc] init]**
- **[MTLDevice newDepthStencilStateWithDescriptor]**

### _sg_mtl_destroy_pipeline()

- 'defer release' for MTLRenderPipelineState
- 'defer release' for MTLDepthStencilState

### _sg_mtl_create_pass()
### _sg_mtl_destroy_pass()

- no Metal functions called here

### _sg_mtl_begin_pass()

- if first pass in frame:
    - wait until oldest frame in flight has finished: **dispatch_semaphore_wait()**
    - **[MTLCommandQueue commandBufferWithUnretainedReferences]**
    - **[MTLBuffer contents]** (get base pointer to this frame's uniform buffer)
- get a new render pass descriptor:
    - if default-pass, call user-callback which might do a:
        - **[MTKView currentRenderPassDescriptor]**
    - or if offscreen pass:
        - **[MTLRenderPassDescriptor renderPassDescriptor]**
- **[MTLCommandBuffer renderCommandEncoderWithDescriptor:]**
- bind the same per-frame uniform buffer to the shader stages (4 slots per stage):
    - 4x **[MTLRenderCommandEncoder setVertexBuffer:offset:atIndex:]**
    - 4x **[MTLRenderCommandEncoder setFragmentBuffer:offset:atIndex:]**

### _sg_mtl_end_pass()

- **[MTLRenderCommandEncoder endEncoding]**

### _sg_mtl_commit()

- **[MTLBuffer didModifyRange:]** (only on macOS to flush uniform buffer content)
- call user callback to get drawable, which might do a:
    - **[MTKView currentDrawable]**
- **[MTLCommandBuffer presentDrawable:]**
- **[MTLCommandBuffer addCompletedHandler:]**
    - in the handler callback: **dispatch_semaphore_signal()**
- **[MTLCommandBuffer commit]**
- at this point, 'garbage-collect' objects in the deferred-release-queue
  which are old enough that the GPU definitely doesn't use them anymore

### _sg_mtl_apply_viewport()

- **[MTLRenderCommandEncoder setViewport:]**

### _sg_mtl_apply_scissor_rect()

- **[MTLRenderCommandEncoder setScissorRect:]**
- it's notable here that the scissor rect is clipped to the frame boundaries first,
Metal doesn't allow parts of the scissor rects to be 'outside'

### _sg_mtl_apply_pipeline()

- only if the pipeline has changed:
    - **[MTLRenderCommandEncoder setBlendColorRed:green:blue:alpha:]**
    - **[MTLRenderCommandEncoder setCullMode:]**
    - **[MTLRenderCommandEncoder setFrontFacingWinding:]**
    - **[MTLRenderCommandEncoder setStencilReferenceValue:]**
    - **[MTLRenderCommandEncoder setDepthBias:slopeScale:clamp:]**
    - **[MTLRenderCommandEncoder setRenderPipelineState:]**
    - **[MTLRenderCommandEncoder setDepthStencilState:]**
- (some finer-grained state-filtering would probably make sense here)

### _sg_mtl_apply_bindings()

- for each vertex buffer in the binding:
    - if binding changed:
        - **[MTLRenderCommandEncoder setVertexBuffer:offset:atIndex]**
- for each vertex stage image in the binding:
    - if binding changed:
        - **[MTLRenderCommandEncoder setVertexTexture:atIndex:]**
        - **[MTLRenderCommandEncoder setVertexSamplerState:atIndex:]**
- for each vertex stage image in the binding:
    - if binding changed:
        - **[MTLRenderCommandEncoder setFragmentTexture:atIndex:]**
        - **[MTLRenderCommandEncoder setFragmentSamplerState:atIndex:]**

### _sg_mtl_apply_uniforms()

- just a memcpy() into the current frame's uniform buffer, and record the offset:
    - **[MTLRenderCommandEncoder setVertexBufferOffset:atIndex:]** OR
    - **[MTLRenderCommandEncoder setFragmentBufferOffset:atIndex:]**

### _sg_mtl_draw()

- **[MTLRenderCommandEncoder drawIndexPrimitives:...]** OR
- **[MTLRenderCommandEncoder drawPrimitives:...]**

### _sg_mtl_update_buffer()

- just a memcpy()
- ...and only on macOS: **[MTLBuffer didModifyRange:]**

### _sg_mtl_update_image()

- for each face (1 or 6):
    - for each mip surface:
        - for each 3D- or array-slice:
            - **[MTLTexture replaceRegion:...]**


...and that's it for the sokol-gfx Metal backend, and for now also for this
little blog post series. Once the new WebGPU backend looks somewhat stable
there will be another Backend Tour of course :)
