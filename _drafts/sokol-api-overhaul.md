---
layout: post
title: Upcoming API changes in the Sokol headers (Feb 2021)
---

In a few days I will merge the next API-breaking update for the sokol
headers. In general I try to keep such breaking updates to a minimum (at most
two or three a year), that's also why the changes in this merge will be a bit
bigger than usual. Since I was going to break compatibility anyway, I added a
few more changes that I had been collecting in the back of my head over the
last 6 months or so.

The changes cover 3 major areas:

1. Making the C APIs more 'language binding friendly', I've written about
this in [my last blog
post](https://floooh.github.io/2020/08/23/sokol-bindgen.html). In parallel to
the API changes I've brought the [automatically generated Zig bindings]
(https://github.com/floooh/sokol-zig/) into a state where they're actually
useful (also see my little [Pacman.zig toy
project](https://github.com/floooh/pacman.zig)).

2. Add a long-missing feature to allow multiple-rendertarget-rendering with
differing pixel formats, and while at it, per-attachment color-write-masks
and blend-state (the latter is currently only supported on the D3D11
and Metal backends though).

3. Some general cleanup in public structs, renaming some items, reorganising
some nested structs for more convenience and clarity.

I tried to make all required changes actual compile errors (which explains
some renames that might seem random at first). When compiling in C mode
(instead of C++), you should also check for any warnings. Further down,
this blog post contains simple "change recipes" for how to replace old
code with new code.

The changes are tracked in the following pull-requests:

- for the sokol headers: https://github.com/floooh/sokol/pull/458
- for the sokol samples: https://github.com/floooh/sokol-samples/pull/80
- for the sokol-shdc tool: https://github.com/floooh/sokol-tools/pull/47

# The Change List

Here's a raw list of the public API changes. Don't worry if this looks a bit overwhelming,
below that will be the concrete change recipes, and below that a small "Q&A" about
the reasons behind the changes.

- sokol_gfx.h:
    - misc changes:
        - **sg_range**: a new struct to hold a pointer/size pair, and helper macros **SG_RANGE** and **SG_RANGE_REF**
        - **sg_color**: a new struct to hold an RGBA color value
        - **sg_features.mrt_independent_blend_states**: a new feature flag to indicate support for per-color-attachment blend states
        - **sg_features.mrt_independent_write_mask**: ditto to indicate if per-color-attachment write-masks are supported
        - in **sg_color_mask** all possible bit-flag combinations now have an
        explicit enumeration value (for instance **SG_COLORMASK_RGA**), this allows to treat sg_color_mask
        as a "proper enum" instead of a "flag-bits enum"
        - clean up sign conversions warnings (-Wsign-conversion on gcc and clang)
    - **sg_pass_action**:
        - **sg_color_attachment_action.val** has been renamed to **.value** and its type changed from **float[4]** to **sg_color**
        - **sg_depth_attachment_action.val** has been renamed to **.value**
        - **sg_stencil_attachment_action.val** has been renamed to **.value**
    - **sg_buffer_desc**:
        - the type of **sg_buffer_desc.size** has been changed from **int** to **size_t**
        - the pointer **sg_buffer_desc.content** has been replaced with a nested **sg_range** struct
        and renamed to **sg_buffer_desc.data**
    - **sg_image_desc**:
        - the struct **sg_sub_image_content** has been removed (in favour of **sg_range**)
        - the type **sg_image_content** has been renamed to **sg_image_data**, and the nested **subimage** array is now of type **sg_range[][]**
        - **sg_image_desc.content** has been renamed to **sg_image_desc.data**
    - **sg_shader_desc**:
        - the type of **sg_shader_uniform_block_desc.size** has been changed from int to size_t
        - **sg_shader_image_desc.type** has been renamed to **.image_type**
        - the **sg_shader_stage_desc.byte_code/.byte_size** pointer/size pair has been replaced
        with a nested **sg_range** struct called **bytecode**
    - **sg_pipeline_desc**:
        - the former **sg_stencil_state** struct has been renamed to **sg_stencil_face_state**
        - the struct **sg_depth_stencil_state** has been split into the structs **sg_depth_state** and **sg_stencil_state**
        - member names in the new structs **sg_depth_state** and **sg_stencil_state** have been shortened (remove the now redundant **depth_** and **stencil_** prefixes)
        - the struct **sg_rasterizer_state** has been eliminated
        - the following **sg_rasterizer_state** members have moved directly into **sg_pipeline_desc**: alpha_to_coverage_enabled, cull_mode, face_winding, sample_count
        - ...and the following **sg_rasterizer_state** members have moved into **sg_depth_state**: depth_bias, depth_bias_slope_scale, depth_bias_clamp (and the *depth_* prefix has been removed)
        - a new nested struct **sg_color_state** to describe per-color-attachment pixel format,
        color write mask and blend state
        - **sg_blend_state.color_write_mask** has moved to **sg_color_state.write_mask**
        - **sg_blend_state.color_format** has moved to **sg_color_state.pixel_format**
        - **sg_blend_state.depth_format** has moved to **sg_depth_state.pixel_format**
        - **sg_blend_state.blend_color** has moved to **sg_pipeline_desc.blend_color**, and its type changes from **float[4]** to **sg_color**
        - **sg_blend_state.color_attachment_count** has become **sg_pipeline_desc.color_count**
        - **sg_pipeline_desc.depth_stencil** has been split into **.depth** and **.stencil**
        - **sg_pipeline_desc.blend** has been replaced with an array of **sg_color_state** called **.colors[]**
    - **sg_pass_desc**:
        - the struct **sg_attachment_desc** has been renamed to **sg_pass_attachment_desc**
    - updated function signatures:
        - **sg_update_buffer()** and **sg_append_buffer()**: the *data_ptr* and *data_size* arguments have been replaced with an **sg_range** pointer
        - **sg_update_image()**: the **sg_image_content** type has been renamed to **sg_image_data**
        - **sg_apply_uniforms()**: the *data* and *num_bytes* arguments have been replaced with an **sg_range** pointer
    - new alternative functions which take float- instead of int-arguments (note the 'f' postfix):
        - **sg_begin_passf()**: width and height as floats
        - **sg_apply_viewportf()**: viewport position and size as floats
        - **sg_apply_scissor_rectf()**: scissor rect position and size as floats

- sokol_app.h: no breaking changes, the following alternative functions have been added (note the 'f' postfix):
    - **sapp_widthf()**: returns the framebuffer width as float
    - **sapp_heightf()**: returns the framebuffer height as float

- sokol_gl.h: no breaking changes, the following alternative functions have been added (note the 'f' postfix):
    - **sgl_viewportf()**: viewport position and size as floats
    - **sgl_scissor_rectf()**: scissor rect position and size as floats

- sokol_debugtext.h:
    - a new struct **sdtx_range** and helper macro **SDTX_RANGE()**
    - **sdtx_font_desc_t.ptr** and **.size** has been merged into a nested **sdtx_range** struct with the name **.data**

- sokol_shape.h:
    - a new struct **sshape_range** and helper macro **SSHAPE_RANGE()**
    - **sshape_buffer_item_t**:
        - the struct members **.buffer_ptr** and **.buffer_size** have been into a nested **sshape_range** struct named **.buffer**
        - the types of **.data_size** and **.shape_offset** have been changed from uint32_t to size_t

- sokol-shdc (shader cross-compiler):
    - the code generated by sokol-shdc no longer directly calls sokol_gfx.h functions to get the runtime backend, instead the caller now needs to pass the correct backend into the generated code (see below in 'Change Recipes' section for examples)
    - various fixes in the generated code required for the sokol_gfx.h public struct changes
    - sokol-shdc can now generate code for the [sokol-zig bindings](https://github.com/floooh/sokol-zig/)

# Change Recipes

## sg_apply_uniforms()

Instead of separate pointer- and size-parameters, **sg_apply_uniforms** now takes a single
**sg_range** pointer:

```c
vs_params_t vs_params = /* ... */;

// OLD:
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, &vs_params, sizeof(vs_params));

// NEW (C, with and without SG_RANGE):
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, &SG_RANGE(vs_params));
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, &(sg_range){ vs_params, sizeof(vs_params)});

// NEW (C++, with and without SG_RANGE):
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, SG_RANGE(vs_params));
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, { vs_params, sizeof(vs_params) });
```

For code which must compile both in C and C++, you can use the special *SG_RANGE_REF()* macro:

```c
// this compiles both in C and C++:
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, SG_RANGE_REF(vs_params));
```

[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/cube-sapp.c)

## sg_update_buffer(), sg_append_buffer()

Same as with *sg_apply_uniforms()*, the separate pointer- and size-parameters are now 
a single **sg_range** pointer:

```c
// OLD:
sg_update_buffer(buf, data_ptr, data_size);
sg_append_buffer(buf, data_ptr, data_size);

// NEW (C, with and without SG_RANGE):
sg_update_buffer(buf, &SG_RANGE(data));
sg_append_buffer(buf, &SG_RANGE(data));
sg_update_buffer(buf, &(sg_range){ data_ptr, data_size});
sg_append_buffer(buf, &(sg_range){ data_ptr, data_size});

// NEW (C++):
sg_update_buffer(buf, SG_RANGE(data));
sg_append_buffer(buf, SG_RANGE(data));
sg_update_buffer(buf, { data_ptr, data_size});
sg_append_buffer(buf, { data_ptr, data_size});
```

## sg_update_image()

The image-data type has been renamed from **sg_image_content** to **sg_image_data**, and
is now composed of a 2D array of **sg_range** structs (which means you can use the
SG_RANGE macro if the subimage pixel data is in a C array):

```c
uint32_t pixels[] = { /*...*/ };

// OLD:
sg_update_image(img, &(sg_image_content){
    .subimage[0][0] = {
        .ptr = pixels,
        .size = sizeof(pixels)
    }
});

// NEW
sg_update_image(img, &(sg_image_content){
    .subimage[0][0] = SG_RANGE(pixels)
});
```

## Initializing sg_pass_action structs

Change **.val** to **.value**:

```c
// OLD:
sg_pass_action pass_action = {
    .colors[0] = { .action = SG_ACTION_CLEAR, .val = { 0.0f, 0.0f, 0.0f, 1.0f } }
};

// NEW:
sg_pass_action pass_action = {
    .colors[0] = { .action = SG_ACTION_CLEAR, .value = { 0.0f, 0.0f, 0.0f, 1.0f } }
};
```

The type of clear-color value has changed from *float[4]* to *sg_color*, so the
clear color can now be assigned in a single statement:
```c
sg_color clear_color = { .r=1.0f, .g=0.5f, .b=0.25f, .a=1.0f };
pass_action.colors[0].value = clear_color;
```

[Sample code](https://github.com/floooh/sokol-samples/blob/master/sapp/clear-sapp.c)

## Creating immutable buffers (with initial data)

The common case of creating a buffer from an array of vertices or indices
now looks like this (using the SG_RANGE helper macro):

```c
float vertices[] = { /*...*/ };

// OLD:
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .size = sizeof(vertices),
    .content = vertices
});

// NEW:
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = SG_RANGE(vertices)
});

// ...or without the SG_RANGE() macro:
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = {
        .ptr = vertices,
        .size = sizeof(vertices)
    }
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/quad-sapp.c)


## Creating dynamic buffers (without initial data)

This looks exactly as before:

```c
// OLD+NEW:
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .size = buffer_size,
    .usage = SG_USAGE_STREAM
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/instancing-sapp.c)

## Creating shaders via sokol-shdc

The generated functions which return an initialized **sg_shader_desc** struct
now no longer call directly into sokol-gfx to query the rendering backend,
instead the backend type must now be passed as argument from the outside:

```c
// OLD:
sg_shader shd = sg_make_shader(triangle_shader_desc());

// NEW:
sg_shader shd = sg_make_shader(triangle_shader_desc(sg_query_backend()));
```

[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/triangle-sapp.c)

## Creating shaders without sokol-shdc

If the shader uses textures, change **.type** to **.image_type**:
```c
// OLD:
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .fs.images[0].type = SG_IMAGETYPE_2D,
    /* ... */
});

// NEW:
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .fs.images[0].image_type = SG_IMAGETYPE_2D,
    /* ... */
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/d3d11/texcube-d3d11.c)

If the shader is precompiled (D3D11 or Metal), pass the bytecode as an **sg_range** struct:

```c
// OLD:
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .vs = {
        .byte_code = vs_bytecode_ptr
        .byte_code_size = vs_bytecode_size
    },
    /* ... */
});

// NEW, if bytecode is in a C array:
const uint8_t vs_bytecode[] = { /*...*/ }
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .vs.bytecode = SG_RANGE(vs_bytecode),
    /* ... */
});

// ...with separate pointer and size:
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .vs.bytecode = {
        .ptr = vs_bytecode_ptr,
        .size = vs_bytecode_size
    }
    /* ... */
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/d3d11/binshader-d3d11.c)

## Image Creation

The nested image content struct has been renamed from **.content** to **.data**, and
the type changed from **sg_image_content** to **sg_image_data**:

```c
uint32_t pixels[8][8] = { /*...*/ };

// OLD:
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 8,
    .height = 8,
    .content.subimage[0][0] = {
        .ptr = pixels,
        .size = sizeof(pixels)
    }
});

// NEW:
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 8,
    .height = 8,
    .data.subimage[0][0] = SG_RANGE(pixels)
});

// ...or without SG_RANGE:
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 8,
    .height = 8,
    .data.subimage[0][0] = {
        .ptr = pixels,
        .size = sizeof(pixels)
    }
});
```

## Pipeline Creation

In general:

- no changes in the vertex layout definition (the nested **.layout** struct)
- the nested **.rasterizer** struct has been "dissolved", and its render states have
been moved to the top-level struct and nested structs
- the nested **.depth_stencil** struct has been split into **.depth** and **.stencil**
- the nested **.blend_state** struct item has been replaced with a nested array of **.color** struct items, which define per-color-attachment attributes

### Creating a minimal pipeline for 3D rendering:

```c
// OLD:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = { /* ... */ },
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .depth_stencil = {
        .depth_compare_func = SG_COMPAREFUNC_LESS_EQUAL,
        .depth_write_enabled = true,
    },
    .rasterizer = {
        .cull_mode = SG_CULLMODE_BACK,
    }
});

// NEW:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = { /* unchanged */ },
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .cull_mode = SG_CULLMODE_BACK,
    .depth = {
        .compare = SG_COMPAREFUNC_LESS_EQUAL,
        .write_enabled = true,
    } 
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/cube-sapp.c)

### Creating a pipeline for alpha-blended rendering:

```c
// OLD:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = { /* ... */ },
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .blend = {
        .enabled = true,
        .src_factor_rgb = SG_BLENDFACTOR_SRC_ALPHA,
        .dst_factor_rgb = SG_BLENDFACTOR_ONE_MINUS_SRC_ALPHA,
        .color_write_mask = SG_COLORMASK_RGB,
    }
});

// NEW:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = { /* ... */ },
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .colors[0] = {
        .write_mask = SG_COLORMASK_RGB,
        .blend = {
            .enabled = true,
            .src_factor_rgb = SG_BLENDFACTOR_SRC_ALPHA,
            .dst_factor_rgb = SG_BLENDFACTOR_ONE_MINUS_SRC_ALPHA,
        }
    }
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/html5/imgui-emsc.cc)

### Offscreen rendering with non-default pixel format and multi-sampling:

```c
// OLD
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = /* ... */
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .depth_stencil = {
        .depth_compare_func = SG_COMPAREFUNC_LESS_EQUAL,
        .depth_write_enabled = true,
    },
    .blend = {
        .color_format = SG_PIXELFORMAT_RGBA32F,
        .depth_format = SG_PIXELFORMAT_DEPTH,
    },
    .rasterizer = {
        .cull_mode = SG_CULLMODE_BACK,
        .sample_count = 4,
    }
});

// NEW:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc))
    .layout = /* ... */
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .cull_mode = SG_CULLMODE_BACK,
    .sample_count = 4,
    .depth = {
        .compare = SG_COMPAREFUNC_LESS_EQUAL,
        .write_enabled = true,
        .pixel_format = SG_PIXELFORMAT_DEPTH,
    },
    .colors[0].pixel_format = SG_PIXELFORMAT_RGBA32F
});
```

### Multiple-Render-Target rendering with default color pixel formats:

```c
// OLD:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = /* ... */
    .shader = shd,
    .index_type - SG_INDEXTYPE_UINT16,
    .depth_stencil = {
        .depth_compare_func = SG_COMPAREFUNC_LESS_EQUAL,
        .depth_write_enabled = true
    },
    .blend = {
        .color_attachment_count = 3,
        .depth_format = SG_PIXELFORMAT_DEPTH
    },
    .rasterizer = {
        .cull_mode = SG_CULLMODE_BACK,
        .sample_count = 4,
    },
});

// NEW:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = /* ... */
    .shader = shd,
    .index_type - SG_INDEXTYPE_UINT16,
    .cull_mode = SG_CULLMODE_BACK,
    .sample_count = 4,
    .depth = {
        .pixel_format = SG_PIXELFORMAT_DEPTH,
        .compare = SG_COMPAREFUNC_LESS_EQUAL,
        .write_enabled = true,
    },
    .color_count = 3,
});
```

The old **.blend.color_attachment_count** has been renamed to
**.color_count** and now also indicates the number of valid items in the
**.colors** array of **sg_color_state** items which describes
color-attachment attributes. But since all those attributes are initialized
from their default values, only the number of color-attachments needs to be
provided as long as the pass color-attachment images also have been created
with default attributes

[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/mrt-sapp.c)

### Multiple-Render-Target rendering with differing pixel formats

On backends which support MRT rendering (which is "all" except GLES2/WebGL1),
the render targets can now have different pixel formats:

```c
// OLD: this was not supported before

// NEW:
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    .layout = /* ... */
    .shader = shd, 
    .index_type = SG_INDEXTYPE_UINT16,
    .cull_mode = SG_CULLMODE_BACK,
    .depth = {
        .pixel_format = SG_PIXELFORMAT_DEPTH,
        .write_enabled = true,
        .compare = SG_COMPAREFUNC_LESS_EQUAL
    },
    .color_count = 3,
    .colors = {
        [0].pixel_format = SG_PIXELFORMAT_RGBA16F,
        [1].pixel_format = SG_PIXELFORMAT_R32F,
        [2].pixel_format = SG_PIXELFORMAT_RGBA8
    }
});
```

>NOTE that you need to provide a count in this case which seems redundant, because
the count could also be "inferred" from the initialized array items. This is 
because "missing" items will be initialized with their default values, yet
sokol-gfx needs to know how many color attachments the pipeline will be configured
with.

You can also setup an MRT pipeline object which per-color-attachment blend
functions, but this is currently only supported in the D3D11 and Metal backends
(check **sg_query_features().mrt_independent_blend_state** for support):

```c
sg_pipeline pip = sg_make_pipeline(&(sg_pipeline_desc){
    /* ... */
    .color_count = 3,
    .colors = {
        [0] = {
            .pixel_format = SG_PIXELFORMAT_RGBA16F,
            .blend = {
                .enabled = true,
                .src_factor_rgb = SG_BLENDFACTOR_SRC_ALPHA,
                .dst_factor_rgb = SG_BLENDFACTOR_ONE_MINUS_SRC_ALPHA,
            },
        },
        [1] = {
            .pixel_format = SG_PIXELFORMAT_RGBA32F,
            .blend = {
                .enabled = true,
                .src_factor_rgb = /* ... */,
                .dst_factor_rgb = /* ... */,
            },
        },
        [2] = {
            .pixel_format = SG_PIXELFORMAT_RGBA8,
            .blend = {
                .enabled = true,
                /* ... */
            },
        }
    }
});
```

# Q & A

## Why the "range" structs?

Mainly to enable better language bindings to languages which have "proper" array-
and slice-types. But it also makes a lot of sense for C and C++ when a pointer/size
pair can be initialized and passed around as a single item.

## Why the sg_color struct?

Because one can't assign an array to another array of the same size in C and C++:

```c
// this doesn't work:
const float a[4] = { 1.0f, 2.0f, 3.0f, 4.0f };
const float b[4] = a;

// but this works:
const sg_color a = { 1.0f, 2.0f, 3.0f, 4.0f };
const sg_color b = a;
```

## Why are there two sizes in sg_buffer_desc?

The *sg_buffer_desc* struct now has two size items, one in the top-level struct,
and one in the nested sg_range struct:

```c
const sg_buffer_desc desc = {
    .size = ...,
    .data.size = ...,
};
```

This might seem a bit confusing, because there are two different items to 
describe the buffer size...

I actually went back and forth on this a few time until I settled for the current
solution. Think of the top-level .size items as the buffer size, and the
.data.size item as the size of the data that's copied into the buffer on 
creation. Currently there's a restriction that the data size must match the
buffer size, but this might be relaxed in later API versions, allowing the initially
copied data to be smaller than the buffer size (current plan is that this will
be part of a "dynamic resource update overhaul" which makes the whole area of copying
data into and between buffers and images more flexible).

Until then you should only initialize one of the size items:

- for immutable buffers with initial data, initialize the **.data.size** and
**.data.ptr** items, and keep **.size** zero-initialized
- for dynamic buffers (without initial data), initialize only the **.size**
item, and keep the whole **.data** nested struct zero-initialized.

The "default-value fixup" that's happening inside sokol-gfx will then take care
of the rest (e.g. it will copy **.data.size** into **.size** for the 'immutable buffer case')

## Why doesn't the GL backend support per-attachment blend-functions?

This will be fixed at a later time.

Currently in the GL backends, all color attachments must use the same blend function
(or precisely, the blend function from the first color attachment will be used
for all other color attachments of a rendering pass).

Fixing this will require to upgrade the desktop GL backend code to use GL
4.0, because the required function
[glBlendFuncSeparatei()](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glBlendFuncSeparate.xhtml)
is only available since GL 4.0. Such a switch also offers other opportunities to
modernize the GL backend, so it makes more sense to do this in a separate update.

The GLES code paths will most likely not get this feature though because the main purpose
of the GLES path is WebGL support, and WebGL2 is "stuck" at GLES 3.0, while
glBlendFuncSeparatei() was only added in GLES 3.2.

## Why don't the GLES/WebGL backends support different color write masks?

Same reason as above, the required function [glColorMaski](https://www.khronos.org/registry/OpenGL-Refpages/es3/html/glColorMask.xhtml) is only available since GLES 3.2.

(but note that the desktop GL backend supports independent color masks).

## Why has the sg_pipeline_desc struct changed that much

The initial reason why sg_pipeline_desc has changed at all was the addition
of per-color-attachment pixel formats. And since the "dam was
broken" anyway, I added a few more changes that had been rolling around in the
back of my head for a while.

From today's point-of-view it makes little sense to structure the pipeline
creation descriptor structs like "traditional" 3D APIs. For instance in
D3D11, the vertex-layout, depth-stencil-state, rasterizer-state and
blend-state can all be set independently. Metal went a huge step forward to
integrate those states, but still treats the depth-stencil-state separately.

D3D12, Vulkan and WebGPU integrated *all* those separate state objects into a
single immutable pipeline-state object, but the creation-information is still
structured like in the "legacy" APIs. There are nested depth-stencil-,
rasterizer- and blend-state struct, even though technically this doesn't make
much sense because those states can't be changed independently anymore.

Instead in sokol_gfx.h I decided now to change things around in sg_pipeline_desc
so that it looks "nicer" when used with C99 designated initializion. All the
redundant prefixes (like "depth_*" or "stencil_*") are gone and instead the
state names with common prefixes are now grouped into their own nested structs.

Since it doesn't make any technical sense to structure the interior of the
sg_pipeline_desc struct in the "traditional" way, it might just as well be
more convenient by requiring less typing.

## Why no bigger signed- vs unsigned-integer API changes

There are a lot of signed integers in the Sokol APIs that "should" be
unsigned because negative values make no sense for then. Originally I was
planning to do a big signed-vs-unsigned cleanup, but reconsidered after an
enlightening Twitter discussion thread:

https://twitter.com/FlohOfWoe/status/1355839062880477187

The gist is that in a language without robust under/overflow checking it is
better to prefer signed integers even for values that not supposed to be
negative, because it's too easy to produce accidental underflows when
subtracting two unsigned integers, but hard to check whether an unsigned
underflow actually happend (because you don't get a negative number, but a
very big positive number out of it, which might be indistuingishable from 
large valid numbers).

Unsigned integers should only for special cases, like bit-wise operations, or
modulo arithmetics (where overflow is an actual "feature").

I allowed one other exception, when an item is usually assigned from a
**sizeof()** statement: In that case I opted for using the **size_t** type
for that item.

Unfortunately this rule of "almost always signed" isn't a good match for more
strongly typed languages like Zig and Rust, because (for instance) both of
those languages use unsigned integers for common things like array indices.
*But* those languages also have robust over/underflow checking, so using unsigned
integers makes more sense in there. This presents a conflict in the
whole idea to make the Sokol C-APIs more binding-friendly.

Long story short: I'll come up with a separate solution which will resolve
this conflict without plastering the C-API with unsigned integers. One idea
is to introduce specific typedefs in the C-APIs such as
**sg_count**, **sg_index** etc... which then can remain signed on the C-API
side, but the auto-generated bindings may change those to unsigned type.
Currently I'm not a big fan of that idea though because it makes the C-API
more "obscure" (strong-typing fans might disagree). Another solution which I
currently prefer is to use a manually maintained type-override-map to change
the type of specific API items.

Over and out :)
