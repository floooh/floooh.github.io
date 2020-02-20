---
layout: post
title: "sokol_gfx.h Backend Tour: OpenGL"
---

(update: [Part 2](https://floooh.github.io/2020/02/18/sokol-gfx-backend-tour-d3d11.html), [Part 3](https://floooh.github.io/2020/02/20/sokol-gfx-backend-tour-metal.html))

I recently did a bit of code cleanup in the existing sokol-gfx backends as
preparation for the new WebGPU backend (which is currently early work-in-progress).

And with the GL, D3D11 and Metal backends somewhat stable for quite a while now, it's
a good opportunity to have a closer look on what the different backends look
like under the hood and how they differ.

First a general overview:

## How sokol_gfx.h backends are implemented

Backends are entirely 'compile time beasts' integrated with the
backend-agnostic code via conditional compilation. Backend-specific code also
lives entirely on the 'implementation side', meaning the public API is
completely backend-agnostic and bit for bit the same no matter what backend is
selected. 

There are no 'virtual function tables' or 'void pointers' connecting
backend-agnostic data structures with backend-specific data, this 'static
nature' of backends gives the compiler/linker good optimization
opportunities when sokol_gfx.h is linked as a static library, and since the
public API is entirely backend-agnostic, also allows to compile sokol-gfx into
backend-specific DLLs which can be loaded into and used from the same executable.

A new backend is created by implementing a specific set of structs and functions
and then wrapping those structs and functions into general type-
and function names which are then used by the higher-level parts of sokol-gfx.

All structs and functions belonging to a specific backend have a specific
name prefix (this prefix isn't technically necessary but makes the code easier
to read and search):

- OpenGL backend structs and functions start with: *_sg_gl_...*
- D3D11 backend: *_sg_d3d11_...*
- Metal backend: *_sg_mtl_...*
- WebGPU backend (WIP): *_sg_wgpu_...*

A backend must define the following structs (using the GL backend prefixes
as example):

- **_sg_gl_buffer_t**: all state for a buffer object
- **_sg_gl_image_t**: ditto for image object
- **_sg_gl_shader_t**: ...shader object
- **_sg_gl_pipeline_t**: ...pipeline object
- **_sg_gl_pass_t**: ...render pass object
- **_sg_gl_context_t**: ...context object
- plus a special global 'backend state' struct **_sg_gl_backend_t**

The 'resource structs' are typedef'ed to common names used in the
higher-level backend-agnostic code based on the selected backend:

```c
typedef _sg_gl_buffer_t _sg_buffer_t;
typedef _sg_gl_image_t _sg_image_t;
typedef _sg_gl_shader_t _sg_shader_t;
typedef _sg_gl_pipeline_t _sg_pipeline_t;
typedef _sg_gl_pass_t _sg_pass_t;
typedef _sg_gl_context_t _sg_context_t;
```

Those common structs contain both backend-agnostic and backend-specific
data as nested structs:

```c
typedef struct {
    _sg_slot_t slot;
    _sg_buffer_common_t cmn;
    struct {
        // Gl-specific stuff
    } gl;
} _sg_gl_buffer_t;
```

The **slot** struct is the same across all common resource-type structs and
stores resource-pool housekeeping data. The **cmn** struct holds
'backend-agnostic' resource-type-specific data, this is usually needed only for
the validation layer and (currently) dead-weight when the validation layer is
disabled (this will be subject to another code cleanup pass eventually).
Finally the **gl** nested struct stores data that's specific to the GL backend.

Higher up in the backend-agnostic code parts, and after being typedef'ed to a
backend-agnostic name like **_sg_buffer_t**, these resource-structs are
used in resource-pools (which is essentially just a fancy name for a flat
array of those structs).

On to the backend-functions: The idea here is the same as for structs. There's
a specific set of backend-specific functions which are then wrapped in 
common backend-agnostic function names which are called by the higher level
code.

This is the list of functions a backend must implement (again, with the
GL prefixes as examples). Those functions should be quite familiar since they
are equivalent to public API function names:

- **_sg_gl_setup_backend()**
- **_sg_gl_discard_backend()**
- **_sg_gl_reset_state_cache()**
- **_sg_gl_create_context()**
- **_sg_gl_destroy_context()**
- **_sg_gl_activate_context()**
- **_sg_gl_create_buffer()** 
- **_sg_gl_destroy_buffer()**
- **_sg_gl_create_image()**
- **_sg_gl_destroy_image()**
- **_sg_gl_create_shader()**
- **_sg_gl_destroy_shader()**
- **_sg_gl_create_pipeline()**
- **_sg_gl_destroy_pipeline()**
- **_sg_gl_create_pass()**
- **_sg_gl_destroy_pass()**
- **_sg_gl_begin_pass()**
- **_sg_gl_end_pass()**
- **_sg_gl_commit()**
- **_sg_gl_apply_viewport()**
- **_sg_gl_apply_scissor_rect()**
- **_sg_gl_apply_pipeline()**
- **_sg_gl_apply_bindings()**
- **_sg_gl_apply_uniforms()**
- **_sg_gl_draw()**
- **_sg_gl_update_buffer()**
- **_sg_gl_append_buffer()**
- **_sg_gl_update_image()**

There are two small special-case internal helper functions which are needed for 
the backend-agnostic code to pull some data from backend-specific data structures
(this is probably an indication of a small design-wart which should be solved better):

- **_sg_gl_pass_color_image()**
- **_sg_gl_pass_ds_image()**

So all in all, a new sokol-gfx backend must define 7 data structures and 30 functions.

What follows now is a closer look at the GL backend, essentially, how the backend
functions map to GL functions. The GL backend is by far the most complex
sokol-gfx backend, so the following might look a bit overwhelming and boring. It
will all make a bit more sense when being compared to the D3D11 and Metal backends
though.

## How sokol-gfx functions map to GL functions

### _sg_gl_setup_backend()

- depending on the underlying GL API (GLES2/GLES3/GL3.3) available extensions are inspected
(**glGetString(), glGetIntegerv(), glGetStringi()**)
- GL limits are queried (**glGetIntegerv(GL_MAX_TEXTURE_SIZE)**, etc...)

### _sg_gl_discard_backend()

no GL functions are called

### _sg_gl_reset_state_cache()

The following GL functions are called to bring GL and sokol-gfx into a 
defined "default state":

- **glBindVertexArray()** (not on GLES2)
- 2x **glBindBuffer()** to clear the GL_ARRAY_BUFFER and GL_ELEMENT_ARRAY_BUFFER slots
- 12x (SG_MAX_SHADERSTAGE_IMAGES) to clear all texture bindings:
    - **glActiveTexture(GL_TEXTURE0+i)**
    - **glBindTexture(GL_TEXTURE2D, 0)**
    - **glBindTexture(GL_TEXTURE_CUBE_MAP, 0)**
    - not on GLES2: **glBindTexture(GL_TEXTURE_3D)**
    - not on GLES2: **glBindTexture(GL_TEXTURE_2D_ARRAY)**
- up to 16x (for each vertex attribute slot):
    - **glDisableVertexAttribArray()**
- reset depth-stencil state:
    - **glEnable(GL_DEPTH_TEST)**
    - **glDepthFunc(GL_ALWAYS)**
    - **glDepthMask(GL_FALSE)**
    - **glDisable(GL_STENCIL_TEST)**
    - **glStencilFunc(GL_ALWAYS, 0, 0)**
    - **glStencilOp(GL_KEEP, GL_KEEP, GL_KEEP)**
    - **glStencilMask(0)**
- reset blend state:
    - **glDisable(GL_BLEND)**
    - **glBlendFuncSeparate(GL_ONE, GL_ZERO, GL_ONE, GL_ZERO)**
    - **glBlendEquationSeparate(GL_FUNC_ADD, GL_FUNC_ADD)**
    - **glColorMask(GL_TRUE, GL_TRUE, GL_TRUE, GL_TRUE)**
    - **glBlendColor(0.0f, 0.0f, 0.0f, 0.0f)**
- reset rasterizer state:
    - **glDisable(GL_POLYGON_OFFSET_FILL)**
    - **glDisable(GL_CULL_FACE)**
    - **glFrontFace(GL_CW)**
    - **glCullFace(GL_BACK)**
    - **glEnable(GL_SCISSOR_TEST)**
    - **glDisable(GL_SAMPLE_ALPHA_TO_COVERAGE)**
    - **glEnable(GL_DITHER)**
    - **glDisable(GL_POLYGON_OFFSET_FILL)**
    - only on GL3.3: **glEnable(GL_MULTISAMPLE)**
    - only on GL3.3: **glEnable(GL_PROGRAM_POINT_SIZE)**

### _sg_gl_create_context()

- **glGetIntegerv(GL_FRAMEBUFFER_BINDING)**: queries the default framebuffer binding
because this is different from zero on some platforms
- **glGenVertexArrays(1, ..); glBindVertexArray()**: This creates and binds a global
vertex array object, which is required on GLES3 and GL3.3 Core Profile.

### _sg_gl_destroy_context()

- **glDeleteVertexArrays()**: destroys the global VAO (not on GLES2)

### _sg_gl_activate_context()

This calls **_sg_gl_reset_state_cache()** which is necessary to bring sokol-gfx
into a defined state after the GL context has been switched.

### _sg_gl_create_buffer()

To create an immutable buffer with data, the following GL functions are called:

- **glGenBuffers(1, ...)**
- **glBufferData(...)**: without data to allocate underlying storage
- **glBufferSubData(...)**: to copy the actual data into the buffer

For dynamic buffers, the following GL functions are called twice (to enable
double-buffered data updates later):

- **glGenBuffers(1, ...)**
- **glBufferData(...)**: with size but no data to allocate underlying storage

### _sg_gl_destroy_buffer()

For each GL buffer created in _sg_gl_create_buffer(), **glDeleteBuffers()** 
is called.

### _sg_gl_create_image()

- if the image is a depth-stencil-rendertarget texture:
    - **glGenRenderBuffers()**
    - **glBindRenderBuffer()**
    - **glRenderBufferStorage()** OR **glRenderBufferStorageMultisample()**
- otherwise (no depth-stencil rendertarget):
    - if this is a render target, and not GLES2, and multi-sampled:
        - **glGenRenderBuffers()**
        - **glBindRenderBuffer()**
        - **glRenderBufferStorageMultisample()**
    - if this is an immutable image, do the following once, otherwise twice:
        - **glGenTextures(1, ...)**
        - **glActiveTexture() + glBindTexture()**
        - up to 11x **glTexParameter()** for setting various sampler parameters
        - for each face (1 or 6):
            - for each mipmap surface:
                - **glCompressedTexImage2D()** OR
                - **glTexImage2D()** OR
                - **glCompressedTexImage3D()** OR
                - **glTexImage3D()**
        - **glActiveTexture() + glBindTexture()** (to restore original texture binding)

### _sg_gl_destroy_image()

- 1x or 2x: **glDeleteTexture()**
- up to 2x: **glDeleteRenderBuffers()**

### _sg_gl_create_shader()

- 2x for compiling vertex- and fragment-shader:
    - **glCreateShader()**
    - **glShaderSource()**
    - **glCompileShader()**
    - if compilation failed:
        - **glGetShaderiv()**
        - **glGetShaderInfoLog()**
        - **glDeleteShader()**
- **glCreateProgram()**
- 2x **glAttachShader()**
- **glLinkProgram()**
- 2x **glDeleteShader()**
- **glGetProgramiv(..GL_LINK_STATUS..)**
- if failed:
    - **glGetProgramiv()**
    - **glGetProgramInfoLog()**
    - **glDeleteProgram()**
- resolve uniform locations:
    - for each shader stage:
        - for each uniform block in shader stage:
            - for each uniform in uniform block:
                - **glGetUniformLocation()**
- resolve image locations:
    - for each shader stage:
        - for each image in shader stage:
            - **glGetUniformLocation()**

### _sg_gl_destroy_shader()

- **glDeleteProgram()** (that's it)

### _sg_gl_create_pipeline()

If the pipeline's vertex layout contains vertex attribute names, those
are resolved into locations here:

- up to 16x **glGetAttribLocation()**

...and that's it.

### _sg_gl_destroy_pipeline()

...nothing here.

### _sg_gl_create_pass()

- **glGetIntegerv(GL_FRAMEBUFFER_BINDING,...)** to store the current frame buffer binding
- **glGenFramebuffer()**
- **glBindFramebuffer(GL_FRAMEBUFFER, ...)**
- if this is an MSAA pass:
    - for each color attachment:
        - **glFramebufferRenderbuffer()**
- otherwise (not an MSAA pass):
    - for each color attachment, depending on attachment type:
        - **glFramebufferTexture2D()** OR
        - **glFramebufferTextureLayer()**
- if a depth-stencil-attachment exists:
    - **glFramebufferRenderbuffer()** for the depth-attachment
    - and optionally **glFramebufferRenderbuffer()** for the stencil-attachment
- **glCheckFramebufferStatus()** to make sure the framebuffer is complete
- if this is an MSAA pass, create 'MSAA resolve buffers':
    - for each color attachment:
        - **glGenFramebuffers(1, ..)**
        - **glBindFramebuffer()**
        - depending on attachment type:
            - **glFramebufferTexture2D()** OR
            - **glFramebufferTextureLayer()**
        - **glCheckFramebufferStatus()** to check if framebuffer is complete
- finally **glBindFramebuffer()** to restore the original framebuffer binding

### _sg_gl_destroy_pass():

- up to 6x **glDeleteFramebuffer()**
        
### _sg_gl_begin_pass()

- **glBindFramebuffer()**
- if not default pass, and not GLES2: **glDrawBuffers()** (for MRT rendering)
- **glViewport()** to reset viewport to framebuffer size
- **glScissor()** same for scissor rect
- if necessary (check state cache), enable various states before clearing the framebuffer:
    - **glColorMask(GL_TRUE, GL_TRUE, GL_TRUE, GL_TRUE)**
    - **glDepthMask(GL_TRUE)**
    - **glDepthFunc(GL_ALWAYS)**
    - **glStencilMask(0xFF)**
- if not GLES2 and MRT-pass (multiple-render-target):
    - for each color attachment:
        - **glClearBufferfv(GL_COLOR, ...)** (only if SG_ACTION_CLEAR requested)
    - if depth-stencil-attachments exist, and depending on SG_ACTION_CLEAR on those:
        - **glClearBufferfv(GL_DEPTH_STENCIL, ...)** OR
        - **glClearBufferfv(GL_DEPTH, ...)** OR
        - **glClearBufferuiv(GL_STENCIL, ...)**
- otherwise (all depending on whether SG_ACTION_CLEAR is requested):
    - **glClearColor()**
    - **glClearDepth()** OR **glClearDepthf()**
    - **glClearStencil()**
    - **glClear()**

### _sg_gl_end_pass()

- if not GLES2, and not default pass, and an MSAA pass, do an MSAA-resolve blit:
    - **glBindFramebuffer(GL_READ_FRAMEBUFFER, ...)**
    - for each color attachment:
        - **glBindFramebuffer(GL_DRAW_FRAMEBUFFER, ...)**
        - **glReadBuffer()**
        - **glDrawBuffers()**
        - **glBlitFramebuffer()**
- bind the default framebuffer: **glBindFramebuffer()**

### _sg_gl_apply_viewport()

- **glViewport()** (that's it)

### _sg_gl_apply_scissor_rect()

- **glScissor()** (that's it)

### _sg_gl_apply_pipeline()

Apply new GL state filtered through the GL state cache:

- **glDepthFunc()**
- **glDepthMask()**
- **glEnable/glDisable(GL_STENCIL_TEST)**
- **glStencilMask()**
- 2x (front/back face) **glStencilFuncSeparate()**
- 2x (front/back face) **glStencilOpSeparate()**
- **glEnable/glDisable(GL_BLEND)**
- **glBlendFuncSeparate()**
- **glBlendEquationSeparate()**
- **glColorMask()**
- **glBlendColor()**
- **glDisable(GL_CULL_FACE)** OR **glEnable(GL_CULL_FACE) + glCullFace()**
- **glFrontFace()**
- **glEnable/glDisable(GL_SAMPLE_ALPHA_TO_COVERAGE)**
- **glEnable/glDisable(GL_MULTISAMPLE)** (only on GL3.3)
- **glPolygonOffset()**
- **glEnable/glDisable(GL_POLYGON_OFFSET_FILL)**
- **glUseProgram()**

### _sg_apply_bindings()

- for each shader stage:
    - for each image bound to shader stage:
        - **glUniform1i()**
        - **glActiveTexture()**
        - **glBindTexture()**
- only if changed: **glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ...)**
- for each vertex attribute, and only if changed:
    - if vertex attribute enabled:
        - **glBindBuffer(GL_ARRAY_BUFFER, ...)**
        - **glVertexAttribPointer()**
        - **glVertexAttribDivisor()** (if instancing supported)
        - **glEnableVertexAttribArray()**
    - otherwise (vertex attribute disabled)
        - **glDisableVertexAttribArray()**

### _sg_apply_uniforms()

- for each uniform in uniform block, depending on uniform type:
    - **glUniform1fv()** OR
    - **glUniform2fv()** OR
    - **glUniform3fv()** OR
    - **glUniform4fv()** OR
    - **glUniformMatrix4fv()**

Note that each call to **_sg_apply_uniform()** will result in a single
call to **glUniform4fv()** if "flattened uniform blocks" are used via
a shader compiler like [sokol-shdc](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md).

### _sg_draw()

One of:

- **glDrawElements()** OR
- **glDrawElementsInstanced()** OR
- **glDrawArrays()** OR
- **glDrawArraysInstanced()**

### _sg_gl_commit()

...clear any 'left-over' buffer- and texture-bindings:

- up to 2x **glBindBuffer()**
- up to 12x **glActiveTexture()**
- up to 48x **glBindTexture()**

### _sg_gl_update_buffer(), _sg_gl_append_buffer()

- **glBindBuffer()** (filtered by state cache)
- **glBufferSubData()**
- **glBindBuffer()** (to restore previous buffer binding, filtered by state cache)

### _sg_gl_update_image()

- **glActiveTexture() + glBindTexture()** (filtered by state cache)
- for each face (1 or 6):
    - for each mipmap surface:
        - **glTexSubImage2D()** OR
        - **glTexSubImage3D()**
- **glActiveTexture() + glBindTexture()** (to restore previous texture binding, filtered by state cache)

## The End

...and that's it for the GL backend. As you can see, GL can be a very verbose and messy API :/

The next two blog posts in the series will do the same 'under the hood' look
for the D3D11 and Metal backends, and those will be a lot shorter :)
