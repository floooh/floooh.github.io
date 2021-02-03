---
layout: post
title: The Great API Overhaul (Feb 2021)
---

In a few days I will do the next API breaking merge for the sokol headers. In general
I try to keep such breaking changes to a minimum (at most two or three a year),
that's also why the changes in this merge will be a bit bigger than usual. Since
I was going to break compatibility anyway, I added a few more changes that I
had been collecting in the back of my head over the last 6 months or so.

The changes cover 3 areas:

1. Making the C APIs more 'language binding friendly', I've written about
this in [my last blog
post](https://floooh.github.io/2020/08/23/sokol-bindgen.html). In parallel to
the API changes I've brought the [automatically generated Zig bindings]
(https://github.com/floooh/sokol-zig/) into a state where they're actually
useful (also see my little [Pacman.zig toy
project](https://github.com/floooh/pacman.zig)).

2. Add a long-missing feature to allow pass-color-attachments with different
pixel formats, and while at it, per-attachment color-write-masks and blend-state
(although the latter is currently only supported on the D3D11 and Metal backends).

3. Some general cleanup in public structs, renaming some items, reorganising
some nested structs for more convenience and "clarity", and so on...

I tried to make all required changes actual compile errors (which might explain
some "non-sensical" renames). But at least when compiling in C mode (instead of C++),
you also need to check for warnings. I'll try to provide simple "recipes" how
to replace old code with new code in this blog post though.

# The Change List

Here's a raw list of the public API changes. Don't worry if this looks a bit overwhelming, below
that will follow some general explanation what motivated those changes, and concrete "change recipes"
describing how to change your existing code.

- sokol_gfx.h:
    - misc changes:
        - **sg_range**: a new struct to hold a pointer/size pair, and helper macros **SG_RANGE** and **SG_RANGE_REF**
        - **sg_color**: a new struct to hold an RGBA color value
        - **sg_features.mrt_independent_blend_states**: a new feature flag to indicate support for per-color-attachment blend states
        - **sg_features.mrt_independent_write_mask**: ditto to indicate if per-color-attachment write-masks are supported
        - in **sg_color_mask** all possible bit-flag combinations now have an explicit
        enumeration value (for instance **SG_COLORMASK_RGA**)
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
        - the following **sg_rasterizer_state** members have moved directly into **sg_pipeline_desc*: alpha_to_coverage_enabled, cull_mode, face_winding, sample_count
        - ...and the following **sg_rasterizer_state** members have moved into **sg_depth_state**: depth_bias, depth_bias_slope_scale, depth_bias_clamp (the depth_ prefix has been removed)
        - a new nested struct **sg_color_state** to describe per-color-attachment pixel format,
        color write mask and blend state
        - **sg_blend_state.color_write_mask** has moved to **sg_color_state.write_mask**
        - **sg_blend_state.color_format** has moved to **sg_color_state.pixel_format**
        - **sg_blend_state.depth_format** has moved to **sg_depth_state.pixel_format**
        - **sg_blend_state.blend_color** has moved to **sg_pipeline_desc.blend_color**, and its type changes from **float[4]** to **sg_color**
        - **sg_blend_state.color_attachment_count** has become **sg_pipeline_desc.color_count**
        - **sg_pipeline_desc.depth_stencil** has been split into **.depth** and **.stencil**
        - **sg_pipeline_desc.blend** has been replaced with a nested array of sg_color_state called **.colors[]**
    - **sg_pass_desc**:
        - the struct **sg_attachment_desc** has been renamed to **sg_pass_attachment_desc**
    - updated function signatures:
        - **sg_update_buffer()** and **sg_append_buffer()**: the *data_ptr* and *data_size* arguments have been replaced with an **sg_range** pointer:
        - **sg_update_image()**: the **sg_image_content** type has been renamed to **sg_image_data**
        - new function **sg_begin_default_passf()** which takes the *width* and *height* arguments as float
        - new function **sg_apply_viewportf()** which takes float arguments
        - new function **sg_apply_scissor_rectf()** which takes float arguments
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
    - a new struct **sdtx_range** and helper macro **SDTC_RANGE()**
    - **sdtx_font_desc_t.ptr** and **.size** has been merged into a nested **sdtx_range** struct with the name **.data**

- sokol_shape.h:
    - a new struct **sshape_range** and helper macro **SSHAPE_RANGE()**
    - **sshape_buffer_item_t**:
        - the struct members **.buffer_ptr** and **.buffer_size** have been into a nested **sshape_range** struct named **.buffer**
        - the types of **.data_size** and **.shape_offset** have been changed from uint32_t to size_t

- sokol-shdc (shader cross-compiler):
    - the code generated by sokol-shdc no longer directly calls sokol_gfx.h functions to get the runtime backend, instead the caller now needs to pass the correct backend into the generated code (see below in 'Change Recipes' section for examples)
    - fixes in the generated code required for the sokol_gfx.h public struct changes
    - sokol-shdc can now generate code for the [sokol-zig bindings](https://github.com/floooh/sokol-zig/)

# Change Recipes

## Initializing sg_pass_action structs

Change **.val** to **.value**:

```c
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
// create a vertex buffer from a C array
float vertices[] = { /*...*/ };
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = SG_RANGE(vertices)
});

// ...or create an index buffer from a C array
uint16_t indices[] = { /*...*/ };
sg_buffer ibuf = sg_make_buffer(&(sg_buffer_desc){
    .type = SG_BUFFERTYPE_INDEXBUFFER,
    .data = SG_RANGE(indices)
});
```

[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/quad-sapp.c)

...or if you prefer to not use the SG_RANGE() macro:
```c
// create a vertex buffer from a C array
float vertices[] = { /*...*/ };
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = {
        .ptr = vertices,
        .size = sizeof(vertices)
    }
});

// ...or create an index buffer from a C array
uint16_t indices[] = { /*...*/ };
sg_buffer ibuf = sg_make_buffer(&(sg_buffer_desc){
    .type = SG_BUFFERTYPE_INDEXBUFFER,
    .data = {
        .ptr = indices,
        .size = sizeof(indices)
    }
});
```

...the SG_RANGE() macro is also useless if you have a pointer and size of the
data:
```c
sg_buffer create_vertex_buffer(const void* data_ptr, size_t data_size) {
    return sg_make_buffer(&(sg_buffer_desc){
        .data = {
            .ptr = data_ptr,
            .size = data_size
        }
    });
}
```

## Creating dynamic buffers (without initial data)

This looks exactly as before:

```c
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .size = buffer_size,
    .usage = SG_USAGE_STREAM
});
```
[Sample Code](https://github.com/floooh/sokol-samples/blob/master/sapp/instancing-sapp.c)

## Creating shaders via sokol-shdc

You need to pass the sokol-gfx backend into the generated functions which return
**sg_shader_desc** structs (note the call to *sg_query_backend()*:

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
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .fs.images[0].image_type = SG_IMAGETYPE_2D,
    /* ... */
});
```

[Sample Code](https://github.com/floooh/sokol-samples/blob/master/d3d11/texcube-d3d11.c)

If the shader is precompiled (D3D11 or Metal), pass the bytecode as an **sg_range** struct:

```c
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .vs.bytecode = { .ptr = bytecode_ptr, .size = bytecode_size }
    /* ... */
});
```

...or if the bytecode is in C array, you can also use the SG_RANGE helper macro:

```c
const uint8_t vs_bytecode[] = { /*...*/ }
sg_shader shd = sg_make_shader(&(sg_shader_desc){
    /* ... */
    .vs.bytecode = SG_RANGE(vs_bytecode)
    /* ... */
});
```

[Sample Code](https://github.com/floooh/sokol-samples/blob/master/d3d11/binshader-d3d11.c)

## Pipeline Creation

In general:

- no changes in the vertex layout definition (the nested **.layout** struct)
- the nested **.rasterizer** struct has been disolved, and its render states have
been distributed to the top-level or other nested structs
- the nested **.depth_stencil** strust has been split into **.depth** and **.stencil**
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

- FIXME: pipeline with alpha-blending
- FIXME: MRT pipeline with default attachment params
- FIXME: MRT pipeline with differing pixel formats
- FIXME: MRT with differing color write mask and blend state



# General non-breaking changes

sokol_gfx.h, sokol_app.h and sokol_gl.h gained a couple new functions which 
take float parameters instead of integers. This may reduce noisy casting between
floats and integers in languages which don't implicitly convert between the two
(and may also get rid of some casts needed to supress compiler-specific warnings
in C and C++).

Those new functions which take float parameters are named the same as their
integer counterparts but with an 'f' postfix. They accept the same value-ranges
as their integer counterparts. Other then that the float-functions behave the
same as the integer-functions. Float values will be clamped to integer, there
is no "subpixel-functionality". 

The new functions are:

In **sokol_gfx.h**:
```c
void sg_begin_default_passf(const sg_pass_action* pass_action, float width, float height);

void sg_apply_viewportf(float x, float y, float width, float height, bool origin_top_left);

void sg_apply_scissor_rectf(float x, float y, float width, float height, bool origin_top_left);
```

In **sokol_app.h**:
```c
float sapp_widthf(void);

float sapp_heightf(void);
```

In **sokol_gl.h**:
```c
sgl_viewportf(float x, float y, float w, float h, bool origin_top_left);

sgl_scissor_rectf(float x, float y, float w, float h, bool origin_top_left);
```

# Introducing range structs (pointer/size pairs)

So far, binary data has been passed into the APIs as a pointer and a separate
size in bytes (e.g. two separate struct members, or two separate function
parameters). This separation has now been removed and pointer/size pairs are now
bundled in "range-structs". This has advantages both for working in C, and in
languages which have "proper" array and slice types.

The advantage for working in C is that pointers to data and the associated
data size is now initialized and passed around as one item. This makes it
harder for the size to get mixed up with something else or get lost completely.

Many new languages have "proper" array slices which bundle a base pointer and
length. Having a corresponding type in the C API makes it easier for language
bindings to connect to those language-native types.

Range structs have been added to the following headers so far:

- sokol_gfx.h: **sg_range**
- sokol_debugtext.h: **sdtx_range**
- sokol_shape.h: **sshape_range**

A range struct looks like this (with the sokol_gfx.h sg_range struct as example,
for other headers, use the respective prefix):

```c
typedef struct sg_range {
    const void* ptr;
    size_t size;
} sg_range;
```

If you have a pointer and "dynamic" size, you'd usually create the range struct
like this:

```c
const sg_range range = { .ptr = my_pointer, .size = my_size };
```

...or this is fine too:

```c
const sg_range range = { my_pointer, my_size };
```

For the common situation where you have a fixed size array, like an array of
vertices or indices, you can use the **SG_RANGE()** helper macro:

```c
const float vertices[] = { 0.0f,0.0f, 1.0f,0.0f, 1.0f,1.0f, 0.0f,1.0f };
const sg_range range = SG_RANGE(vertices);
```

**SG_RANGE** simply expands to:
```c
#define SG_RANGE(x) (sg_range){ &x, sizeof(x) }
```
...when compiled as C, and:
```cpp
#define SG_RANGE(x) sg_range{ &x, sizeof(x) }
```
...when compiled as C++.


# Change-recipes for range structs

## sokol_gfx.h

Creating a vertex buffer with immutable data:

***OLD***
```c
float vertices[] = { /*...*/ };

sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .size = sizeof(vertices),
    .content = vertices
});
```
***NEW***
```c
float vertices[] = { /*...*/ };

sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = SG_RANGE(vertices)
});
```
...or alternatively:
```c
float vertices[] = { /*...*/ };

sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .data = {
        .ptr = vertices,
        .size = sizeof(vertices)
    }
});
```

Note that creating dynamic buffers without initial data *hasn't* changed:
```c
sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
    .usage = SG_USAGE_STREAM,
    .size = buffer_size
});
```
For the obvious question why there are two members describing sizes in sg_buffer_desc, please
check out the Q&A section at the end of this blog post :)

# TODO
- sg_color
- sg_features.mrt_independent_blend_state
- sg_features.mrt_independent_write_masks



# FIXME: Q & A section

- why are there two size members in sg_buffer_desc
