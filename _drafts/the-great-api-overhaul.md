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

# The Changelist

Here's a quick rundown of the changes. Don't worry if this looks a bit overwhelming, below
that will follow some general explanation what motivated those changes, and concrete "change recipes"
describing how to change your existing code.

- sokol_gfx.h:
    - new structs and minor misc stuff:
        - **sg_range**: a new struct to hold a pointer/size pair
        - **sg_color**: a new struct to hold an RGBA color value
        - **sg_features.mrt_independent_blend_states**: a new featur flag to indicate support for per-color-attachment blend states
        - **sg_features.mrt_independent_write_mask**: ditto for allowind per-color-attachment color write masks
        - in **sg_color_mask** all possible bit-flag combinations now have an explicit
        enumeration value (for instance **SG_COLORMASK_RGA**)
    - **sg_pass_action**:
        - in **sg_color_attachment_action.val** has been renamed to **.value** and its type changed from **float[4]** to **sg_color**
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


[TO BE CONTINUED!]


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
