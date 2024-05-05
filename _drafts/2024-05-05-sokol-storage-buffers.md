---
layout: post
title: "The upcoming sokol-gfx storage buffer update (May 2024)"
---

I a couple of days I will merge the next sokol-gfx feature update which adds initial
storage buffer support. The udpate also affects other headers and tools (most notably
sokol_app.h and sokol-shdc - the cross-backend shader compiler).

The bad news first:

- This is 'gpu-readonly' support, e.g. it's not possible to populate storage buffers
  with the GPU, this will come in a future 'compute shaders' update.
- The following platform/backend combos don't get storage buffer support:
    - all GLES3 backends (WebGL2, iOS, Android): for WebGL2 and iOS there is no
      other choice since they are stuck with GLES 3.0, for Android, storage buffer
      support may be added at a later point
    - macOS+GL: macOS is stuck at GL 4.1, while storage buffers require at least
      GL 4.3
- This leaves the following platform/backend combos which support storage buffers:
    - macOS + Metal
    - iOS + Metal
    - Windows + D3D11
    - Windows + GL
    - Linux + GL
    - Web + WebGPU

In general, storage buffers provide a convenient way to communicate
large array-like data to the CPU, for instance:

- use 'vertex pulling' to load per-vertex and/or per-instance data from
  storage buffers instead of relying on the fixed function vertex input
  stage
- as a more more convenient and flexible way to load random access
  structured data in shaders compared to the old-school way of using
  'data textures'.

...and as a 'drive-by' feature: sokol-gfx now finally allows to kick off
a draw call without any resource bindings and instead synthesize vertices
'out of thin air' in the vertex shader.

## How to check for storage buffer support

The struct `sg_features` has a new item `bool storage_buffer`. To check for storage
buffer support:

```c
if (sg_query_features().storage_buffer) {
    // storage buffers are supported...
} else {
    // storage buffers are *NOT* supported...

}
```

## Desktop GL version caveats (and a minor breaking change)

The sokol_gfx.h desktop-GL backend will now query what GL version it runs on
to decide whether storage buffers are supported (storage buffers were added in GL 4.3).

The expected minimal version has been bumped to 4.1 on macOS and 4.3 on other platforms, this
also means that sokol_app.h will now by default create a 4.1 context on macOS, and 4.3 context
on other platforms.

Since the GL version is now flexible, the configuration define `SOKOL_GLCORE33` doesn't make
much sense anymore and has been renamed to `SOKOL_GLCORE`. You'll get a proper compile
error when trying to build with the old `SOKOL_GLCORE33` define.

Apart from rebuilding your shaders via an updated sokol-shdc, this is the only *required*
change for existing code.

In sokol-shdc, the target language `glsl330` has been removed and replaced
with `glsl410` and `glsl430`. When targeting the macOS GL backend, use `glsl410`,
otherwise `glsl430`.

## Cross-backend shader authoring with sokol-shdc

Let's rewrite the [cube-sapp.glsl](https://github.com/floooh/sokol-samples/blob/master/sapp/cube-sapp.glsl) shader
to pull vertices from a storage buffer instead of the fixed function vertex input.

The original shader declares the vertex input with vertex attributes:

```glsl
in vec4 position;
in vec4 color0;
```

> NOTE: the cube-sapp.glsl shader makes use of a fixed function vertex input feature which extends float[3] vertex input to a vec4 with a w-component of 1.0, magic like this isn't supported when
using storage buffers


For vertex pulling this is replaced with a flexible-array struct inside a buffer interface block.

```glsl
struct sb_vertex {
    vec3 pos;
    vec4 color;
};

readonly buffer ssbo {
    sb_vertex vtx[];
};
```

> NOTE: I'm using `sb_vertex` for the struct name here because `vertex` is a reserved keyword
in the Metal Shading Language and would cause a compile error when outputing MSL.

Do not use an attribute like `layout(std430, binding=0)` for the buffer interface block,
sokol-shdc will take of those details.

The original vertex shader looks like this:

```glsl
void main() {
    gl_Position = mvp * position;
    color = color0;
}
```

Converted to vertex pulling it looks like this:

```glsl
void main() {
    gl_Position = mvp * vec4(vtx[gl_VertexIndex].pos, 1.0);
    color = vtx[gl_VertexIndex].color;
}
```

Note how `gl_VertexIndex` (not `gl_VertexID`!) is used to index into the storage buffer,
this is because sokol-shdc shaders are written in 'Vulkan style', not 'GL style'.

We also need to expand the vec3 input pos manually to a vec4 with w-component = 1.0.

That's all changes needed on the shader side. Next compile the modified shader with:

```sh
sokol-shdc -i shader.glsl -o shader.h -l metal_macos:hlsl5:glsl430:wgsl -f sokol
```

Apart from the 'traditional' code-generation output, sokol-shdc will create two new
declarations:

- A define `#define SLOT_ssbo (0)`, this is the bind slot index to be used in the `sg_bindings` struct

- A C struct `sb_vertex_t` which maps the GLSL struct `sb_vertex` to the C side looking like this:

    ```c
    SOKOL_SHDC_ALIGN(16) typedef struct sb_vertex_t {
        float pos[3];
        uint8_t _pad_12[4];
        float color[4];
    } sb_vertex_t;
    ```

> NOTE: with the right `@ctype` tags at the top of the shader we could
also map the struct members to C or C++ types, for instance with HandmadeMath.h types:

```c
SOKOL_SHDC_ALIGN(16) typedef struct sb_vertex_t {
    hmm_vec3 pos;
    uint8_t _pad_12[4];
    hmm_vec4 color;
} sb_vertex_t;
```

Next let's see how the [cube-sapp C code](https://github.com/floooh/sokol-samples/blob/master/sapp/cube-sapp.c) needs to be changed:

The original code creates a vertex buffer like this:

```c
float vertices[] = {
    -1.0, -1.0, -1.0,   1.0, 0.0, 0.0, 1.0,
     1.0, -1.0, -1.0,   1.0, 0.0, 0.0, 1.0,
    ...
};
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = SG_RANGE(vertices),
    .label = "cube-vertices"
});
```

By default `sg_make_buffer()` creates a vertex buffer, so the above is
identical with a more explicit:

```c
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .type = SG_BUFFERTYPE_VERTEXBUFFER,
    .data = SG_RANGE(vertices),
    .label = "cube-vertices"
});
```

...when changing the code to use storage buffers we can use the
code-generated `sb_vertex_t` struct to initialize the vertex data.
This has the advantage that we don't need to care about the obscure
`std430` memory layout rules for adding the correct amount of padding bytes:

```c
sb_vertex_t vertices[] = {
    { .pos = { -1.0, -1.0, -1.0 }, .color = { 1.0, 0.0, 0.0, 1.0 } },
    { .pos = {  1.0, -1.0, -1.0 }, .color = { 1.0, 0.0, 0.0, 1.0 } },
    ...
};
sg_buffer sbuf = sg_make_buffer(&(sg_buffer_desc){
    .type = SG_BUFFERTYPE_STORAGEBUFFER,
    .data = SG_RANGE(vertices),
    .label = "cube-vertices",
});
```

...note how the buffer type has changed to `SG_BUFFERTYPE_STORAGEBUFFER`.

On to the `sg_pipeline` object. In the original code, a vertex layout
must be defined in the `sg_pipeline_desc` struct to configure the
fixed function vertex input stage:

```c
state.pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = {
        .attrs = {
            [ATTR_vs_position].format = SG_VERTEXFORMAT_FLOAT3,
            [ATTR_vs_color0].format   = SG_VERTEXFORMAT_FLOAT4
        }
    },
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .cull_mode = SG_CULLMODE_BACK,
    .depth = {
        .write_enabled = true,
        .compare = SG_COMPAREFUNC_LESS_EQUAL,
    },
    .label = "cube-pipeline"
});
```

When pulling vertex data from storage buffers such a vertex layout description isn't needed:

```c
state.pip = sg_make_pipeline(&(sg_pipeline_desc){
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .cull_mode = SG_CULLMODE_BACK,
    .depth = {
        .write_enabled = true,
        .compare = SG_COMPAREFUNC_LESS_EQUAL,
    },
    .label = "cube-pipeline"
});
```

...and finally the original call to `sg_apply_bindings()`:

```c
sg_apply_bindings(&(sg_bindings){
    .vertex_buffers[0] = vbuf,
    .index_buffer = ibuf,
});
```

...is changed like this:

```c
sg_apply_bingings(&(sg_bindings){
    .index_buffer = ibuf,
    .vs.storage_buffers[SLOT_ssbo] = sbuf,
});
```

...and that's it! On the CPU side, storage buffers actually simplify a lot of code because
you don't need a vertex layout in the `sg_pipeline_desc` struct, and you get a properly
aligned and padded C struct for the storage buffer content.

> NOTE: A 'proper' cross-backend sample should also check whether storage buffers are
actually supported via `sg_query_features().storage_buffer` and render some
sort of fallback.

## Shader Authoring Caveats

[TODO]

## Under the hood

[TODO]

## Using storage buffers without sokol-shdc

[TODO]