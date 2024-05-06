---
layout: post
title: "Upcoming Sokol header API changes (May 2024)"
---

Aka: "the storage buffer update"

In a couple of days I will merge the next sokol-gfx feature update which adds
initial storage buffer support. The update also affects other headers and tools
(most notably sokol_app.h, all headers with embedded shaders, and sokol-shdc -
the cross-backend shader compiler).

The bad news first:

- This is 'gpu-readonly' support, e.g. it's not possible (yet) to write to storage buffers
  from shader code, gpu-write support will come in a future 'compute shaders' update.
- The following platform/backend combos don't get storage buffer support:
    - all GLES3 backends (WebGL2, iOS+GLES3, Android): for WebGL2 and iOS there is no
      other choice since they are stuck with GLES 3.0, for Android, storage buffer
      support may be added later
    - macOS+GL: macOS is stuck at GL 4.1, while storage buffers require at least
      GL 4.3
- This leaves the following platform/backend combos which support storage buffers:
    - macOS + Metal
    - iOS + Metal
    - Windows + D3D11
    - Windows + GL
    - Linux + GL
    - Web + WebGPU

Storage buffers provide a convenient way to communicate
large array-like data to shaders (the minimum guaranteed size for storage buffers
is 128 MBytes), for instance:

- for 'vertex pulling' to load per-vertex and/or per-instance data from
  storage buffers instead of relying on the fixed function vertex input
  stage
- as a more convenient and flexible way to load random access
  data in shaders compared to the old-school way of using
  'data textures'.

...and as a 'drive-by' feature: sokol-gfx now finally allows to kick off
a draw call without any resource bindings and instead synthesize vertices
'out of thin air' in the vertex shader.

The root PR for the update is here: [#1007](https://github.com/floooh/sokol/pull/1007).


## New sample code

The following backend-agnostic samples have been added (those use sokol_app.h and sokol-shdc).

> NOTE: You'll need a recent Chrome for the WebGPU sample links to work, also expect
some general breakage and rendering artifacts depending on the platform (for
instance I see pixel noise artifacts in the `sbuftex-sapp` sample on my Windows
PC with an NVIDIA RTX 2070, and Chrome on Android straight up crashes the tab
on most samples). Also please note that the source code links in those samples
will not be valid until all the update PRs have been merged.

- **triangle-bufferless-sapp**: this demonstrates rendering without buffers (and
  is the only new sample that also works on backends without storage buffer support):
    - WebGPU: [triangle-bufferless-sapp.html](https://floooh.github.io/sokol-webgpu/triangle-bufferless-sapp.html)
    - C code: [sapp/triangle-bufferless-sapp.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/triangle-bufferless-sapp.c)
    - GLSL code: [sapp/triangle-bufferless-sapp.glsl](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/triangle-bufferless-sapp.glsl)
- **vertexpull-sapp**: the cube-sapp sample ported to vertex pulling:
    - WebGPU: [vertexpull-sapp.html](https://floooh.github.io/sokol-webgpu/vertexpull-sapp.html)
    - C code: [sapp/vertexpull-sapp.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/vertexpull-sapp.c)
    - GLSL code: [sapp/vertexpull-sapp.glsl](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/vertexpull-sapp.glsl)
- **sbuftex-sapp**: a sample which uses a storage buffer in the fragment shader stage:
    - WebGPU: [sbuftex-sapp.html](https://floooh.github.io/sokol-webgpu/sbuftex-sapp.html)
    - C code: [sapp/sbuftex-sapp.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/sbuftex-sapp.c)
    - GLSL code: [sapp/sbuftex-sapp.glsl](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/sbuftex-sapp.glsl)
- **instancing-pull-sapp**: vertex pulling and instancing via storage buffers:
    - WebGPU: [instancing-pull-sapp.html](https://floooh.github.io/sokol-webgpu/instancing-pull-sapp.html)
    - C code: [sapp/instancing-pull-sapp.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/instancing-pull-sapp.c)
    - GLSL code: [sapp/instancing-pull-sapp.glsl](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/instancing-pull-sapp.glsl)
- **ozz-storagebuffer-sapp**: the ozz-skin sample rewritten to pull vertex-, instance- and skinning-matrices from storage buffers:
    - WebGPU: [ozz-storagebuffer-sapp.html](https://floooh.github.io/sokol-webgpu/ozz-storagebuffer-sapp.html)
    - C code: [sapp/ozz-storagebuffer-sapp.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/ozz-storagebuffer-sapp.c)
    - GLSL code: [sapp/ozz-storagebuffer-sapp.glsl](https://github.com/floooh/sokol-samples/blob/storage-buffers/sapp/ozz-storagebuffer-sapp.glsl)

The following backend-specific samples demonstrate how to use storage buffers without the sokol-shdc shader compiler:

- **D3D11** [d3d11/vertexpulling-d3d11.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/d3d11/vertexpulling-d3d11.c)
- **Metal**: [metal/vertexpulling-metal.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/metal/vertexpulling-metal.c)
- **WebGPU**: [wgpu/vertexpulling-wgpu.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/wgpu/vertexpulling-wgpu.c)
- **desktop GL**: [glfw/vertexpulling-glfw.c](https://github.com/floooh/sokol-samples/blob/storage-buffers/glfw/vertexpulling-glfw.c)


## How to check for storage buffer support

To check for storage buffer support at runtime, call `sg_query_features()` and check the `storage_buffer` boolean in the result:

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

## A simple vertex pulling example

First let's rewrite the [cube-sapp.glsl](https://github.com/floooh/sokol-samples/blob/master/sapp/cube-sapp.glsl) shader
to pull vertices from a storage buffer instead of the fixed function vertex input.

The original shader declares the vertex input with vertex attributes:

```glsl
in vec4 position;
in vec4 color0;
```

> NOTE: the cube-sapp.glsl shader makes use of a fixed function vertex input
feature which extends float[3] vertex data on the CPU side to vec4 with a w-component 1.0
on the GPU side. Magic like this isn't supported when reading from storage buffers (as far as
I'm aware at least).

For vertex pulling the input vertex attributes are replaced with a flexible-array struct inside a buffer interface block.

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
in the Metal Shading Language and would cause a compile error when outputting MSL.

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
    vec4 position = vec4(vtx[gl_VertexIndex].pos, 1.0);
    gl_Position = position;
    color = vtx[gl_VertexIndex].color;
}
```

Note how `gl_VertexIndex` (not `gl_VertexID`!) is used to index into the storage buffer,
this is because sokol-shdc shaders are written in 'Vulkan style', not 'GL style'.

We also need to expand the vec3 input pos manually to a vec4 with w-component = 1.0.

That's all the changes needed on the shader side. Next compile the modified shader with:

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
`std430` memory layout rules:

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

When pulling vertex data from storage buffers such a vertex layout description isn't needed, so
the pipeline creation can be simplified to this:

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

...the original `sg_bindings` struct that's passed into `sg_apply_bindings()`:

```c
state.bind = (sg_bindings) {
    .vertex_buffers[0] = vbuf,
    .index_buffer = ibuf
};
```

...is changed like this:

```c
state.bind = (sg_bindings) {
    .index_buffer = ibuf
    .vs.storage_buffers[SLOT_ssbo] = sbuf,
};
```

...and that's it! On the CPU side, storage buffers actually simplify a lot of code because
you don't need a vertex layout in the `sg_pipeline_desc` struct, and you get a properly
aligned and padded C struct for the storage buffer content from sokol-shdc.

> NOTE: A 'proper' cross-backend sample should also check whether storage buffers are
actually supported via `sg_query_features().storage_buffer` and render some
sort of fallback.

## Shader Authoring Caveats

Shader authoring via sokol-shdc is a bit more restricted than vanilla GLSL:

1. A storage buffer interface block must contain exactly one item, and this
  item must be a flexible struct array member. In vanilla GLSL you can have
  additional 'header items' in front of the flexible array member, but this
  turned out tricky to map to CPU-side non-C languages that don't allow
  flexible array members (I actually need to research the various target languages a bit more, maybe
  this rule can be relaxed in the future for some of the target languages).
2. Currently the following types are valid inside a storage buffer struct:
    - `bool, bvec2..4`: mapped to int32_t, and int32_t[2..4]
    - `int, ivec2..4`: mapped to int32_t, and int32_t[2..4]
    - `uint, uvec2..4`: mapped to uint32_t, and uint32_t[2..4]
    - `float, vec2..4`: mapped to float and float[2..4]
    - `matNxM` where N=2..4 and M=1..4 mapped to float[2..64]
3. nested structs
4. arrays of the above

Please note that only few of those combinations are tested, especially when it
comes to correct array item padding and alignment. If you stumble over any problems
please write a ticket at [https://github.com/floooh/sokol-tools/issues](https://github.com/floooh/sokol-tools/issues).

To load packed vertex components from storage buffers, use the following GLSL builtins:

- `vec2 unpackUnorm2x16(uint p)`
- `vec2 unpackSnorm2x16(uint p)`
- `vec4 unpackUnorm4x8(uint p)`
- `vec4 unpackSnorm4x8(uint p)`

## Under the hood

> NOTE: the following information about shader bind slots are only relevant if
you do not use the sokol shader compiler (sokol-shdc), but instead pass 'raw'
HLSL, MSL, GLSL or WGSL shaders into sokol_gfx.h. Also, this information will
become obsolete/irrelevant with another future update I have in mind which will allow
more flexibility when mapping sokol-gfx bind slots to backend 3D API bind slots
(see this planning ticket for more info: [#1037](https://github.com/floooh/sokol/issues/1037))

### Metal

On Metal there is no 'buffer zoo' like in other 3D APIs, uniform-, vertex-,
index- and storage-buffer are the same buffer objects. The vertex-
and fragment-shader stages have their own bind slot spaces though.

The following bind slot ranges are used for the various sokol-gfx
buffer types:

- on the vertex shader stage:
    - **`slots 0..3`** for uniform buffer bindings (sokol-gfx internally
      manages an uniform buffer which might be bound at up to four different
      offsets)
    - **`slots 4..11`** for vertex buffer bindings
    - **`slots 12..19`** for storage buffer bindings
- on the fragment shader stage:
    - **`slots 0..3`** for uniform buffer bindings
    - **`slots 4..11`** for storage buffer bindins

When authoring Metal shaders directly you'll need to use the above bind slots
(also see low-level [Metal backend samples](https://github.com/floooh/sokol-samples/tree/master/metal).

### D3D11

On D3D11, so called [*Byte Address
Buffers*](https://learn.microsoft.com/en-us/windows/win32/direct3d11/overviews-direct3d-11-resources-intro#raw-views-of-buffers)
are used for storage buffers which makes their direct usage in manually written
HLSL a bit awkward (but is not an issue when using sokol-shdc).

If this turns out to be a problem I might add D3D11-specific creation flags to
`sg_buffer_desc` to allow using different D3D11 buffer and buffer-view types
under the hood, details like this might also change again once compute shader
support is added.

On D3D11 and HLSL storage buffers share a bind slot range with texture bindings, that's
why sokol-gfx defines the following bind ranges for textures and storage buffers in
HLSL:

- **`register(t0..t15)`**: reserved for texture bindings
- **`register(t16..t23)`**: reserved for storage buffer bindings

Also see the low-level [D3D11 backend samples](https://github.com/floooh/sokol-samples/tree/master/d3d11) for details.


### WebGPU

Storage buffers are created with `WGPUBufferUsage_Storage`. WebGPU uses a common bind slot
space across all shader resource types and shader stages. Sokol-gfx reserves the following bind
slot ranges for the different shader stages and resource types, use those when feeding manually
written WGSL shaders into sokol-gfx:

- vertex shader stage:
    - textures: **`@group(1) @binding(0..15)`**
    - samplers: **`@group(1) @binding(16..31)`**
    - storage buffers; **`@group(1) @binding(32..47)`**
- fragment shader stage:
    - textures: **`@group(1) @binding(48..63)`**
    - samplers: **`@group(1) @binding(64..79)`**
    - storage buffers: **`@group(1) @binding(80..95)`**

Also see the low-level [WebGPU backend samples](https://github.com/floooh/sokol-samples/tree/storage-buffers/wgpu) for details

### GL

In GL, storage buffers are bound to the `GL_SHADER_STORAGE_BUFFER` target. Sokol-gfx
does not lookup GLSL storage buffer interface blocks by name, but instead expects that
the GLSL code that's passed into `sg_make_shader()` uses a `layout(std430, binding=N)`
annotation to define the bind slot.

The vertex- and fragment-shader stage use a common bind space:

- on the vertex shader stage, use **`binding 0..7`**
- on the fragment shader stage, use **`binding 7..15`**

Also see the low-level [desktop GL backend samples](https://github.com/floooh/sokol-samples/tree/storage-buffers/glfw) for details.

## sokol-shdc updates

Sokol-shdc has been massively refactored, mainly with the goal to have a more
robust base for extracting reflection information from shaders and a more
'structured' approach to code generation so that supporting additional CPU-side
languages will be easier in the future (I'm not yet sure if that last goal was actually achieved though, but time will tell).

Unfortunately this massive refactoring also means that there's a possibility that new
bugs have sneaked in. If you notice anything weird, please write tickets here:

[https://github.com/floooh/sokol-tools/issues](https://github.com/floooh/sokol-tools/issues).

A couple of unrelated lingering bugs have been fixed as well:

- C++ exceptions are now enabled and exceptions coming out of SPIRVCross are now caught
  and turned into proper error messages. Previously sokol-shdc would simply appear to
  crash if SPIRVCross emitted an error (because without C++ exceptions enabled,
  those errors would be turned into a panic which looks like a segfault).
- Error and warning line numbers had been off by a couple of lines recently.
  This has been fixed and error messages now point to the correct line again.
- A couple of somewhat esoteric code generation bugs in non-C code generators
  were fixed (but as I said, it's also quite likely that I have introduced new bugs
  in that area, since code generators were completely rewritten)

## What's next:

In short:

- A resource binding cleanup (see
  [#1037](https://github.com/floooh/sokol/issues/1037)), the main motivation
  for this is that the `sg_bindings` struct is growing quite large and would
  grow even larger if a new compute shader stage is added. Furthermore, the artificial
  separation of shader stages when binding resources also doesn't map
  particularly well to some modern 3D APIs.
- After that it's finally time to tackle compute shaders. For this I need to
  come up with a resource synchronization strategy, but I will most likely
  just copy what WebGPU does.

But first I will probably take a little break and dabble a bit with Zig and emulator coding :)
