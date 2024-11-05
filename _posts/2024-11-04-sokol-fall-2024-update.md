---
layout: post
title: "Upcoming Sokol header API changes (Nov 2024)"
---

In a couple of days I will merge the next breaking sokol_gfx.h update (aka the
"Bindings Cleanup"). The update also affects sokol-shdc, so if you're using
sokol-shdc for shader compilation make sure to update that as well.

* TOC
{:toc}

## Overview

In general, the update makes the relationship between the shader resource interface
and the sokol-gfx resource binding model more explicit, but also more flexible.
Another motivation for the change was to prepare the sokol-gfx API for compute
shader support.

The root PR is here: [https://github.com/floooh/sokol/pull/1111](https://github.com/floooh/sokol/pull/1111).

The TL;DR is:

- When using sokol-shdc for shader compilation, the input GLSL source
  now requires explicit binding annotations via `layout(binding=N)`, where
  `N` directly maps to bindslot indices in the sokol-gfx resource binding API.
- The concept of 'shader stages' mostly disappears from the sokol-gfx API,
  shader stages are now only a minor detail of the shader interface reflection
  information in the `sg_shader_desc` struct passed into the `sg_make_shader()`
  function.
- When *not* using sokol-shdc there's now an explicit mapping from sokol-gfx bindslots
  to 3D backend-specific bindslots. This reduces the sokol-gfx internal
  magic for mapping the backend-agnostic sokol-gfx binding model to the specific binding
  models of the backend 3D APIs (there *are* still some restrictions but only
  when they allow a more efficient resource binding implementation in sokol-gfx).

In general, all changes result in compile errors, and cleaning up the
compile errors by following the 'change recipes' below should be enough
to make your existing code work.

The following parts of the public sokol_gfx.h API have changed:

- In the `sg_bindings` struct, the nested vertex- and fragment-stage structs
  for the image-, sampler- and storage-buffer-bindings have been removed,
  and the bindings arrays have moved up into the root struct.
- In the `sg_apply_uniforms()` call, the shader stage parameter has been removed
- The interior of the `sg_shader_desc` struct and the typename of nested structs
  have changed completely (but if you are using sokol-shdc for shader authoring
  you don't need to worry about that, since sokol-shdc will code-generate
  the `sg_shader_desc` struct.
- A number of public API constants have been removed or renamed (but those
  should rarely show up in user code).
- The enum items in `sg_shader_stage` have been renamed, and those are now
  only used in the `sg_shader_desc` struct and nowhere else:
    - `SG_SHADERSTAGE_VS` => `SG_SHADERSTAGE_VERTEX`
    - `SG_SHADERSTAGE_FS` => `SG_SHADERSTAGE_FRAGMENT`

The update also has some minor behaviour changes:

- Resource bindings can now have gaps, and validation for `sg_apply_bindings()`
  has been relaxed to allow bindslots in the `sg_bindings` struct to be occupied
  even when the current shader doesn't use those bindings. This allows to use
  the same `sg_bindings` struct for different but related shader variants.
- Likewise, uniform block bindslots can now be explicitly defined in the shaders
  which allows to 'share' bindslot indices across shaders. Trying to call
  `sg_apply_uniforms()` for a bindslot that isn't used by the current shader
  is still an error though (not sure yet if this makes sense, could probably
  be relaxed in a later update)
- There's now a new (debug-mode only) error check in `sg_draw()` to make sure
  that `sg_apply_bindings()` and/or `sg_apply_uniforms()` had been called since the
  last `sg_apply_pipeline()` when required.

## Updated documentation and example code

> NOTE: these links will only be uptodate after [PR #1111](https://github.com/floooh/sokol/pull/1111) has been merged.

### When using sokol-shdc:

Please re-read the sokol-shdc documentation:

[https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md](https://github.com/floooh/sokol-tools/blob/master/docs/sokol-shdc.md)

Especially the section `Shader Authoring Considerations`.

In the [sokol_gfx.h header](https://github.com/floooh/sokol/blob/master/sokol_gfx.h), re-read the documentation header above
the `sg_bindings` struct.

Check the updated sokol samples here:

[https://github.com/floooh/sokol-samples/tree/master/sapp](https://github.com/floooh/sokol-samples/tree/master/sapp)

### When *not* using sokol-shdc

In the [sokol_gfx.h header](https://github.com/floooh/sokol/blob/master/sokol_gfx.h), re-read the updated documentation section `ON SHADER CREATION`.

Next read the updated documentation above the `sg_shader_desc` and `sg_bindings` structs.

Finally check the updated backend-specific samples:

- for Metal: [https://github.com/floooh/sokol-samples/tree/master/metal](https://github.com/floooh/sokol-samples/tree/master/metal)
- for D3D11: [https://github.com/floooh/sokol-samples/tree/master/d3d11](https://github.com/floooh/sokol-samples/tree/master/d3d11)
- for desktop GL: [https://github.com/floooh/sokol-samples/tree/master/glfw](https://github.com/floooh/sokol-samples/tree/master/glfw)
- for WebGL/GLES3: [https://github.com/floooh/sokol-samples/tree/master/html5](https://github.com/floooh/sokol-samples/tree/master/html5)
- for WebGPU: [https://github.com/floooh/sokol-samples/tree/master/wgpu](https://github.com/floooh/sokol-samples/tree/master/wgpu)

Especially note the `sg_shader_desc` struct interiors in the `sg_make_shader()` calls.

## Change Recipes

General rule of thumb: fix all places that throw compile errors and
you should be good.

### When using sokol-shdc:

First you'll need to fix your shaders and add explicit binding annotations. When running
sokol-shdc over your current shader code you'll get errors looking like this:

    error: 'binding' : uniform/buffer blocks require layout(binding=X)

...or this:

    error: 'binding' : sampler/texture/image requires layout(binding=X)

To fix those errors for the different resource types add `layout(binding=N)` annotations:

```glsl
layout(binding=0) uniform vs_params { ... };
layout(binding=0) uniform texture2D tex;
layout(binding=0) uniform sampler smp;
layout(binding=0) readonly buffer ssbo { ... };
```

Note that each resource type (uniform blocks, textures, samplers and storage buffers)
has its own bindslot space which is shared across shader stages. Trying to use
bindslot indices outside those ranges, or using the same bindslot for a resource
type in different shader stages will cause a compilation error.

The binding ranges per resource type are:

- uniform blocks: 0..7
- textures: 0..15
- samplers: 0..15
- storage buffers: 0..7

...these are also the maximum number of resources of that type that can be bound
on a shader across all shader stages.

Next fix the compile errors on the CPU side, you should see errors
when initializing an `sg_bindings` struct, when calling `sg_apply_uniforms()`
and possibly when setting up vertex attributes in the `sg_pipeline_desc`
struct:

- in the `sg_bindings` struct, the nested structs for the vertex
  and fragment shader stage have been removed, and the former per-stage
  binding arrays have moved up into the root
- in the `sg_apply_uniforms()` call, the shader stage argument has been removed
- all code-generated slot constants have new naming schemes (also the vertex
  attribute slot constants)

For instance if your shader resource interface looks like this:

```glsl
@vs
// a vertex shader uniform block
layout(binding=0) uniform vs_params { ... };
// a vertex shader texture and sampler
layout(binding=0) uniform texture2D vs_tex;
layout(binding=0) uniform sampler vs_smp;
// a vertex shader storage buffer
layout(binding=0) readonly buffer vs_ssbo { ... };
@end

@fs
// a fragment shader uniform block
layout(binding=1) uniform fs_params { ... };
// diffuse, normal and specular textures
layout(binding=1) uniform texture2D diffuse_tex;
layout(binding=2) uniform texture2D specular_tex;
layout(binding=3) uniform texture2D normal_tex;
// a common sampler for the above textures
layout(binding=1) uniform sampler smp;
@end
```

...the matching `sg_bindings` struct on the CPU side needs to look like
this - note how the array indices match the shader `layout(binding=N)`:

```c
const sg_bindings bnd = {
    .vertex_buffer[0] = ...,
    .index_buffer = ...,
    .images = {
        [0] = vs_tex,
        [1] = diffuse_tex,
        [2] = specular_tex,
        [3] = normal_tex,
    },
    .samplers = {
        [0] = vs_smp,
        [1] = smp,
    },
    .storage_buffers[0] = {
        [0] = vs_ssbo,
    },
};
```

...and the `sg_apply_uniforms()` calls to write the uniform data for the
`vs_params` and `fs_params` uniform blocks now look like this:

```c
sg_apply_uniforms(0, &SG_RANGE(vs_params));
sg_apply_uniforms(1, &SG_RANGE(fs_params));
```

...instead of hardwired numeric indices you can also use code-generated constants
(note that those have been renamed from a generic `SLOT_*` to a per-resource-type
naming scheme):

```c
const sg_bindings bnd = {
    .vertex_buffer[0] = ...,
    .index_buffer = ...,
    .images = {
        [IMG_vs_tex] = vs_tex,
        [IMG_diffuse_tex] = diffuse_tex,
        [IMG_specular_tex] = specular_tex,
        [IMG_normal_tex] = normal_tex,
    },
    .samplers = {
        [SMP_vs_smp] = vs_smp,
        [SMP_smp] = smp,
    },
    .storage_buffers[0] = {
        [SBUF_vs_ssbo] = vs_ssbo,
    },
};
```

...or for the uniform block updates:

```c
sg_apply_uniforms(UB_vs_params, &SG_RANGE(vs_params));
sg_apply_uniforms(UB_fs_params, &SG_RANGE(fs_params));
```

...using the code-generated constants has the advantage that changing the
bindslots in the shader code doesn't require updating the CPU-side code, but other
then that it's totally fine to use numeric indices.

The naming scheme for the code-generated vertex attribute slots has changed
to use the shader program name for 'namespacing' instead of the vertex shader
snippet name.

For instance with the following shader fragment:

```glsl
@vs vs
in vec4 position;
in vec4 color0;
...
@end

@fs fs
...
@end

@program cube vs fs
```

The generated vertex attribute slot constants `ATTR_*` previously looked like this
(in the sg_pipeline_desc struct):

```c
const sg_pipeline_desc desc = {
    .layout = {
        .attrs = {
            [ATTR_vs_position].format = ...,
            [ATTR_vs_color0].format = ...,
        },
    },
    ...
};
```

...now the `ATTR_*` names look like this (e.g. `ATTR_vs_*` to `ATTR_cube_*`):

```c
const sg_pipeline_desc desc = {
    .layout = {
        .attrs = {
            [ATTR_cube_position].format = ...,
            [ATTR_cube_color0].format = ...,
        },
    },
    ...
};
```

...it's also possible to use explicit attribute locations and ignore
the code-generated constants, for instance:

```glsl
@vs vs
layout(location=0) in vec4 position;
layout(location=1) in vec4 color0;
...
@end
```

```c
const sg_pipeline_desc desc = {
    .layout = {
        .attrs = {
            [0].format = ...,
            [1].format = ...,
        },
    },
    ...
};
```

...note though that it's still not allowed to have gaps in the vertex
attribute slots (this may be supported at a later time).

### When *not* using sokol-shdc:

The interior of `sg_shader_desc` has changed to match the new
'shader-stage-agnostic' sokol-gfx binding model. The toplevel-structure
now looks like this:

```c
const sg_shader_desc desc = {
    .vertex_func = { ... },         // vertex shader source or bytecode
    .fragment_func = { ... },       // fragment shader source or bytecode
    .attrs = { ... },               // vertex attribute reflection info
    .uniform_blocks = { ... },      // reflection info for uniform block bindings
    .storage_buffers = { ... },     // reflection info for storage buffer bindings
    .images = { ... },              // reflection info for texture bindings
    .samplers = { ... },            // reflection info for sampler bindings
    .image_sampler_pairs = { ... }, // how images and samplers are used together in the shader
};
```

The array indices in the `uniform_blocks[]` array match the `ub_slot` parameter
in the `sg_apply_uniforms()` call:

    sg_shader_desc.uniform_blocks[N] => sg_apply_uniforms(N, ...)

The array indices in the `storage_buffers[]`, `images[]` and `samplers[]` arrays
match the respective indices in the `sg_bindings` struct:

    sg_shader_desc.images[N] => sg_bindings.images[N]
    sg_shader_desc.samplers[N] => sg_bindings.samplers[N]
    sg_shader_desc.storage_buffers[N] => sg_bindings.storage_buffers[N]

Fields that are only required for a specific 3D backend now have consistent
prefixes:

- D3D11/HLSL: `hlsl_*`
- GL/GLSL: `glsl_*`
- Metal/MSL: `msl_*`
- WebGPU/WGSL: `wgsl_*`

The resource binding slots now require two new types of information:

- the shader stage this resource binding appears on
- a 3D backend specific bindslot

The backend specific bindslot struct members need to be filled with the
shader language specific resource bindslot numbers which also need to
lie within specific ranges:

- for uniform block items:
    - `.hlsl_register_b_n = N` => HLSL `register(bN)` where `(N >= 0) && (N < 8)`
    - `.msl_buffer_n = N` => MSL `[[buffer(N)]]` where `(N >= 0) && (N < 8)`
    - `.wgsl_group0_binding_n = N` => WGSL `@group(0) @binding(N)` where `(N >= 0) && (N < 8)`

- for images:
    - `.hlsl_register_t_n = N` => HLSL `register(tN)` where `(N >= 0) && (N < 24)`
    - `.msl_texture_n = N` => MSL `[[texture(N)]]` where `(N >= 0) && (N < 16)`
    - `.wgsl_group1_binding_n = N` => WGSL `@group(1) @binding(N)` where `(N >= 0) && (N < 128)`

- for samplers:
    - `.hlsl_register_s_n = N` => HLSL `register(sN)` where `(N >= 0) && (N < 16)`
    - `.msl_sampler_n = N` => MSL `[[sampler(N)]]` where `(N >= 0) && (N < 16)`
    - `.wgsl_group1_binding_n = N` => WGSL `@group(1) @binding(N)` where `(N >= 0) && (N < 128)`

- for storage buffers:
    - `.hlsl_register_t_n = N` => HLSL `register(tN)` where `(N >= 0) && (N < 24)`
    - `.msl_register_b_n = N` => MSL `[[buffer(N)]]` where `(N >= 8) && (N < 16)`
    - `.wgsl_group1_binding_n = N` => WGSL `@group(1) @binding(N)` where `(N >= 0) && (N < 128)`
    - `.glsl_binding_n = N` => GLSL `layout(binding=N)` where `(N >= 0) && (N < 16)`

These backend-specific bindslots allow a more flexible mapping from the sokol-gfx
resource binding model to the backend 3D-API binding models, but there are still
some restrictions (which typically exist to allow a more efficient resource binding implementation
in sokol_gfx.h):

- in WebGPU/WGSL, all uniform blocks must be in `@group(0)` and all other
  resource types in `@group(1)`
- in Metal/MSL, the `[[buffer(N)]]` slots 0..7 are reserved for uniform blocks,
  and `[[buffer(N)]]` slots 8..15 are reserved for storage buffers

For code examples, check out the backend-specific samples:

- for Metal: [https://github.com/floooh/sokol-samples/tree/master/metal](https://github.com/floooh/sokol-samples/tree/master/metal)
- for D3D11: [https://github.com/floooh/sokol-samples/tree/master/d3d11](https://github.com/floooh/sokol-samples/tree/master/d3d11)
- for desktop GL: [https://github.com/floooh/sokol-samples/tree/master/glfw](https://github.com/floooh/sokol-samples/tree/master/glfw)
- for WebGL/GLES3: [https://github.com/floooh/sokol-samples/tree/master/html5](https://github.com/floooh/sokol-samples/tree/master/html5)
- for WebGPU: [https://github.com/floooh/sokol-samples/tree/master/wgpu](https://github.com/floooh/sokol-samples/tree/master/wgpu)

...and that should be it! Next big thing on the roadmap: compute shader support :)
