---
layout: post
title: A Tour of sokol_gfx.h
---

*Update:* please also read [this update](http://floooh.github.io/2017/08/06/sokol-api-update.html) 
on what has changed since this post was written!

I have started a new project to move the low-level platform-abstraction
parts of Oryol into their own dependency-free, header-only C libraries
called [Sokol](https://github.com/floooh/sokol).

There's a separate repository for [code samples](https://github.com/floooh/sokol-samples),
and an asm.js/wasm [samples webpage](https://floooh.github.io/sokol-html5/index.html).

For now this is just a 3D API wrapper called [sokol_gfx.h](https://github.com/floooh/sokol/blob/master/sokol_gfx.h).

## Project Scope

sokol_gfx.h is a "modern 3D API wrapper" around GLES2/WebGL, GLES3/WebGL2,
GL3.3, D3D11 and Metal especially suited for small web- and native-mobile
projects.

It mostly takes ideas from Metal and D3D11, plus a few ideas from
D3D12/Vulkan, and wraps them into a simple C API, with GLES2 as the lowest
supported platform.

GLES2/WebGL and GLES3/WebGL2 are the 'lead APIs', and asm.js/wasm are the
'lead platforms', followed by iOS with Metal. Those lead APIs and platforms
guide the sokol_gfx API design decisions (e.g. there will be no API features
from Metal or D3D11 exposed which would be costly to emulate on top of
GLES2/WebGL).

## Back-story

Some personal thoughts and opinions which hopefully help to explain why
sokol_gfx looks the way it does. It seems a bit pompous to base the
design of a simple C library on such large-scale factors, see this more
as a sort background noise which subtly influences every little decision ;)

- New compiled, statically-typed languages emerging, mostly built on top of LLVM
  (Kotlin Native and Rust are two examples).
- A 'modern 3D API' WebGL2 successor is coming, I want to be prepared ;)
- The (slow but steady) reconstruction of the web from a publishing platform 
  into an integrated software-distribution- and runtime-platform.
- The realization that modern software development should happen in small 
  teams on small and independent code bases (there's a death-spiral between
  large teams writing large codebases, causing slow development cycles
  and slow and bloated software products)
- My slowly growing frustration of where C++ is heading (further away from
  CPU and memory, and into 'functional la la land'), I'll not give up on C++
  anytime soon of course, but the language will also not give me anything useful
  in the foreseeable future (at least until C++20).
- My experience that C libraries are usually much easier to integrate 
  into C++ projects than C++ libraries.
- My first contact with the STB header-only libraries.
- My disappointment with the complexity and verbosity of D3D12 and Vulkan, which
  basically require a dedicated 'rendering API team' inside an 'engine team' 
  (see comment above about 'large teams'), and have designed themselves into 
  the 'AAA games on PC' niche.
- The realization that it doesn't matter how fast new technology is adopted,
  instead it matters how fast (or rather slowly) old technology is abandoned.
  The performance and age-gap between the high-end and the low-end has been
  growing dramatically in the last 15..20 years (since GPUs became cheap, and
  3D-APIs became popular). Today's low-end devices and 3D-APIs will be
  relevant for a LONG time, maybe decades.

So how did those thoughts influence the design of sokol_gfx.h:

- it's written in C and as dependency-free header-only lib to simplify
  integration into projects and with other languages
- it favours a small and easy to use API over 'expliciteness' without
  giving up too much flexibility
- it uses a 'modern 3D API' programming style, which is less verbose
  and less error-prone than the GL programming model
- it is small and 'bloat free', perfect for asm.js/WebAssembly
  (web demos start at 33 kilobytes download size)
- sokol_gfx.h is not a fully integrated cross-platform solution, it follows
  more the 'bring your own engine' philosophy of [BGFX](https://github.com/bkaradzic/bgfx)

Some more motivation for such a 'modern wrapper over old APIs' can be
found in this older blog post:

[A modern 3D API wrapper for WebGL](http://floooh.github.io/2016/10/24/altai.html)

## A Tour of the sokol_gfx.h API

Ok, enough of the philosophical ramblings :)

Before diving into the API it is important to know what sokol_gfx 
does *NOT* provide:

* it will *NOT* create a window or the 3D API context/device (this is
  usually done in user code or via other libs like GLFW)
* it does *NOT* provide a cross-platform shader-programming solution,
  instead it accepts 3D-API-specific shader source- or binary-code
  and depend on higher-level code to provide solutions
  (for instance using shader code generation like Oryol)

In general I'm seeing the Sokol header as low-level building blocks, while
they can be used alone for small demos, they are more useful when they 
are the 'bedrock code' of a higher level project, propbably implemented 
in a higher-level language.

### Resources and Structures

sokol_gfx has 5 "baked resource" types (baked means those resources are
'compiled' into immutable objects)

- **buffer**: vertex- and index-data
- **image**: for textures and render targets
- **shader**: vertex- and fragment-shaders, and shader-parameter declarations
- **pipeline**: vertex-layouts, shader, render states
- **pass**: render passes, actions on render targets (clear, MSAA resolve, ...)

Buffers and images are 'immutable resources' in the sense that their
size and attributes cannot change, their *content* can be updated though.

Apart from those 5 resource types, there are 3 'mutable structures' which 
just group rendering parameters, but are not 'compiled' like their
resource counterparts:

- **sg\_draw\_state**: a C struct with resource-binding-slots to be filled with
  the resources used for the next draw call (one pipeline object, 1..N vertex buffers, an optional
  index buffer, and 0..N texture slots each on the vertex-shader and 
  fragment-shader stage)
- **sg\_pass\_action**: describes what should happen when starting a rendering
  pass (for instance clearing the render targets to a specific color)
- **uniform blocks**: these are user-provided C structures holding shader
  parameters (aka uniforms)

### Function Groups

The API is split into 4 groups of functions:

- **misc stuff**: initialization, teardown, check optional feature support,
  boring and uninteresting, so I'll skip this :)
- **resource management**: create, destroy and update baked resource objects
- **drawing**: everything related to drawing, doh
- **struct initializers**: since C has no constructors, initializer functions
  are needed to initialize structures into a default state

### Resource Management

Resource creation and destruction always follows the same pattern:

- initialize a 'desc' structure with creation parameters
- call a resource creation function with a pointer to the 'desc' structure,
  and get a 32-bit 'resource id' back
- use the 'resource id' for rendering, or creating other resource objects
- finally call a resource destruction function

For a resource type 'XXX' (where XXX stands for 'buffer', 'image', 'shader',
'pipeline' or 'pass') it looks like this:

```c
/* initialize a resource description structure */
sg_XXX_desc desc;
sg_init_XXX_desc(&desc);
/* fill the desc with creation parameters */
...
/* create the resource, get a resource id back */
sg_id res_id = sg_make_XXX(&desc);
/* use the resource for rendering and creating other resources */
...
/* finally destroy the resource */
sg_destroy_XXX(res_id);
```

The *sg\_init\_XXX_desc()* functions will initialize the description
structure to a useful default state, for instance when creating a buffer,
the buffer type will be set to 'vertex buffer' and the usage to 'immutable'.
This way application code must only provide creation parameters that differ
from the default state which saves a lot of code (however there's the 
danger that existing application start to misbehave if the default state
changes... something to keep in mind).

Creating a simple vertex buffer with 3 vertices looks like this for instance:

```c
/* create a vertex buffer with 3 vertices */
float vertices[] = {
  // positions         // colors
  0.0f,  0.5f, 0.5f,   1.0f, 0.0f, 0.0f, 1.0f,
  0.5f, -0.5f, 0.5f,   0.0f, 1.0f, 0.0f, 1.0f,
  -0.5f, -0.5f, 0.5f,  0.0f, 0.0f, 1.0f, 1.0f 
};
sg_buffer_desc buf_desc;
sg_init_buffer_desc(&buf_desc);
buf_desc.size = sizeof(vertices);
buf_desc.data_ptr = vertices;
buf_desc.data_size = sizeof(vertices);
sg_id buf_id = sg_make_buffer(&buf_desc);
assert(buf_id);

/* use the buffer somehow... */
...
/* finally destroy the buffer if it is no longer needed */
sg_destroy_buffer(buf_id);
```

This is a good time to mention an important rule when handing pointers
(to data or strings) to sokol_gfx: There are *no* ownership considerations,
sokol will never take ownership of a pointer you provide, it will only
inspect the data and copy what it needs, and it will never modify the data.

Resource creation for the other resource types looks similar, so I won't
repeat the code here, only a list of creation parameters required for each
resource type:

#### Buffer Creation Parameters:

- the size of the buffer in bytes
- the type (vertex or index buffer)
- a usage hint which defines the update strategy (immutable, dynamic or streaming)
- an optional pointer to and size of the initial buffer data

Immutable buffers *must* be initialized with data, for dynamic and streaming
buffers this is optional. The difference between 'dynamic' and 'streaming' 
usage is:

- streaming: the buffer content is updated with new data each frame
- dynamic: the buffer content is updated infrequently (not each frame)

#### Image Creation Parameters:

- the image type: 2D, Cubemap, 3D or Array (3D and Array images are not
  supported on GLES2/WebGL)
- width, height and optionally depth/array layers
- number of mipmaps
- the usage, same as buffers (immutable, dynamic or streaming)
- the pixel format
- texture filter mode (nearest, linear, etc...)
- texture addressing wrap mode (repeat, mirror, clamp)
- whether the image is also a render target
- if the image is a render target, an optional depth/stencil buffer format,
  and an MSAA sample count
- optional data pointers and sizes to fill the image with content

#### Shader Creation Parameters:

For each of the 2 shader-stages (vertex- and fragment-shader-stage):

- shader source- or byte-code
- 0..N uniform block description (bind slot, size, and member layout)
- 0..N image descriptions (bind slot and image type)

The manually provided uniform block and image descriptions are used
for validation checks and to precompute or lookup internal parameters
by the various rendering backends.

There will be different ways to declare uniform blocks, uniform block
members, and images to allow more flexibility with different backend
3D APIs:

- some 3D APIs (like GLES2) can only bind uniform and image samplers
  by their shader variable names
- ...while other 3D APIs and shader languages allow to manually
  declare a bind slot
- for some 3D APIs the internal structure of uniform blocks is irrelevant, only
  their size matters

#### Pipeline Creation Parameters:

Pipeline-state-object creation is where sokol_gfx differs most from GL,
Metal and D3D11, and is more like Vulkan and D3D12:

When creating a pipeline object, the user code must provide:

- all render states (depth-stencil, alpha-blending, rasterizer, all in
all about 25 states, there are no 'free' render states in sokol, except
the scissor- and viewport-rects)
- the 3D primitive type (points, lines, triangles, line-strips or triangle-
strips, this is the common primitive subset supported across all 3D APIs)
- an index data type (none, 16-bit or 32-bit)
- a shader object
- and finally the complete vertex layout:
    - for each vertex buffer bind slot:
        - the vertex stride
        - the vertex-step-mode for instancing (per-vertex, or 
          per-instance, and the step-rate)
        - for each vertex component:
            - the name or attribute bind slot
            - the byte-offset from the start of a vertex
            - the vertex component data type (float, vec2, vec3, ...)

Using pipeline objects on top of GL has 2 advantages:

- there can't be render states 'stuck in the wrong state', since applying
  a pipeline object will reconfigure all render states into the configuration
  defined by the pipeline object
- the GL backend implements its own state cache and will only perform the minimal
  number of GL calls required to transition the GL state machine from its current
  configuration into the new configuration, this is especially useful for 
  WebGL/asm.js/wasm which has a high call overhead

#### Pass Creation Parameters:

Render passes only need to know the render target image ids:

- 1..N color attachment images
- 0..1 depth-stencil attachment images
- a subimage index (which mipmap, cubemap face or 3D/array texture slice to render to)

All images must have been created as render targets, and must have the same
dimensions and MSAA sample count. All color attachments must have the same pixel
format. Some details may change here when the Metal and D3D11 backends are
implemented (this is true for the entire public sokol_gfx API).

### Resource Updates

WebGL and WebGL2 don't have resource mapping functions which would allow
direct access to GPU memory. Instead resource updates must perform a copy from 
existing data in memory. For this reason the resource update model in
sokol_gfx is very simple, but also very restrictive:

There are 2 functions to update the content of buffers and images:

```c
/* update a buffer with new data */
void sg_update_buffer(sg_id buf, const void* data_ptr, int data_size)
/* update an image with new data */
void sg_update_image(sg_id img, int num_data_items, const void** data_ptrs, int* data_sizes)
```

There is only one update allowed per frame and resource object, and data
must be written from the start (but the data size can be smaller than the resource size).

The 3D-API backends take care internally of preventing lock-stalls (that's the main
reason why only one update per resource and frame is allowed, it's the best
compromise to keep the code simple while preventing the user from accidently
triggering a stall, where the CPU must wait for the GPU).

### Drawing Functions

There are only 9 functions related to actual rendering, and most of them
are fairly boring:

```c
void sg_begin_default_pass(const sg_pass_action* pass_action, int w, int h);
void sg_begin_pass(sg_id pass, const sg_pass_action* pass_action);
void sg_apply_viewport(int x, int y, int w, int h, bool origin_top_left);
void sg_apply_scissor_rect(int x, int y, int w, int h, bool origin_top_left);
void sg_apply_draw_state(const sg_draw_state* ds);
void sg_apply_uniform_block(sg_shader_stage stage, int ub_index, const void* data, int num_bytes);
void sg_draw(int base_element, int num_elements, int num_instances);
void sg_end_pass();
void sg_commit();
```

The typical structure of a frame looks like this:

```c
for 1..N sg_begin_pass(pass_id, pass_action);
  for 1..N sg_apply_draw_state(draw_state);
    for 1..N:
      for 0..N sg_apply_uniform_block(shader_stage, slot_index, &ub, sizeof(ub));
      sg_draw(base_element, num_elements, num_instances);
  sg_end_pass();
sg_commit();
```

There are currently 2 variations of *begin\_pass()* depending on whether
rendering should go into render target images (requiring a pass object),
or into the default framebuffer (*sg\_begin\_default\_pass()*).

*sg\_apply\_draw\_state()* takes a pointer to an sg\_draw\_state structure,
this is basically plugging resources into the resource binding slots, and
defines all the resources (pipeline, buffers and images) for the next draw
call. Since *sg\_draw\_state* is just a struct, not a 'baked resource', the
same structure can be 'reslotted' and reused for other calls to
*sg\_apply\_draw\_state()* (the same is true for the *sg\_pass\_action*
structure in *sg\_begin\_pass()*).

*sg\_apply\_uniform\_block()* updates one of the uniform blocks on one
of the 2 shader stages, this also works with a bind-slot model (each shader
stage provides a number of uniform block bind slots). Uniform block updates
are separate from resource binding updates because they usually happen with
different frequency.

There's only a single drawing function *sg\_draw()*, unifying indexed- vs non-indexed and instanced- vs non-instanced rendering. 

*sg\_end\_pass()* finishes the current pass, if the pass was rendering
to an MSAA render target, an MSAA resolve step will happen here.

And finally *sg\_commit()* indicates the end of the current frame.

### Struct Initializers

This is the least interesting group of functions, they only exist because C
doesn't have constructors. You *must* call an initializer function on a C
structure before it can be used to make sure that the structure doesn't
contain random garbage. In debug mode, an 'init guard check' is performed to
ensure that a structure has been initialized (the init functions simply
write a 'magic cookie' value into a special '\_init\_guard' field, which is
then checked by the function which consumes the structure.

## Under The Hood

Some interesting tidbits about the current implementation:

- current line counts (without comments):
    - overall (only GL backend exists so far): 3.2kloc
    - sokol_gfx.h (public types and fwd decls): 0.4kloc
    - GL backend (fairly complete): 1.7kloc
    - backend-agnostic implementation code: 1.1kloc
- there are currently 308 assert checks in the code (~10% of all code)
- the D3D11 and Metal backends will be slightly less code than the GL backend, so I expect
  the overall line count once everything is done to be around 5..6kloc
- sokol_gfx only allocates memory in sg\_setup(), after that
  it is completely allocation-free (of course the underlying 3D API
  will still allocate memory whenever it feels like it)
- all resource objects are kept in pools, each pool does 2 allocations
  when initialized (one for a 'free-slot-queue' and one for the actual
  resource pool)
- the application must define the size of the resource pools when 
  calling sg\_setup() (or just use the default size), the pools cannot be grown later
- the total number of allocations is 11 (1 + 2*num\_resource\_types)
- the allocation size with the default configuration and GL backend
  is around 128 KBytes (other backends will be smaller)
- ...this is for the default pools sizes of 128 buffers, images, pipelines, passes, and 32 shaders)
- the application can provide its own memory allocation, assert and other
  functions by simply defining macros before including the sokol\_gfx header
- in 'declaration mode', sokol\_gfx.h only includes *stdint.h* and *stdbool.h*
- in 'implementation mode', the following headers may also be included:
  *assert.h*, *stdlib.h*, *stdio.h* (only for puts()), and *string.h*, the exact
  includes depend on what 3D backend is used, and what custom functions
  are provided by the application (e.g. if the application provided
  its own assert macro, *assert.h* will not be included)

And this is all for today. The next things in sokol_gfx will be:

- more samples as feature coverage-tests
- make the code more C standard-conforming (I'm not sure yet if I will use C99 or an older standard, I'm tending towards C99)
- Metal and D3D11 backends
- more validation code to catch API usage errors
- better documentation (in old STB tradition this will mostly be extensive 
code comments in the sokol_gfx.h header)
