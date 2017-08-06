---
layout: post
title:  Minor Sokol API Updates
---

I've been integrating some valuable Twitter-verse feedback into 
[sokol_gfx.h](https://github.com/floooh/sokol)
and with this some things I wrote in the [last
post](http://floooh.github.io/2017/07/29/sokol-gfx-tour.html) are no longer
true, here's what has changed:

## No more universal resource id type

The universal resource handle **sg\_id** has changed into
one type per resource: **sg\_buffer**, **sg\_image**, **sg\_shader**, 
**sg\_pipeline** and **sg\_pass**. With this the compiler can now
do proper type checking on resource ids and will complain at compile-time
if the wrong resource type is accidently used somewhere.

## No more dual color/depth-stencil images

Previously it was possible to create an image as render target
which could have both a color-buffer and depth-stencil-buffer,
the same object would then be plugged into a color-attachment-,
and the depth-stencil-attachment-slot of a pass object.

Now an image can either be a color-render-target, or a depth-stencil-
render-target, but not both at the same time. 

## Zero-init and C99 designated initializers

This is the most interesting change:

Previously, desc-structures had to be initialized to a default state
by calling a special initializer function, which basically implemented
the 'missing C++ constructor'.

There are 2 changes related to initialization:

- all initializer functions have been removed
- a zero-initialized structure is now considered to be in 'default state'

This means:

```c
/* instead of this: */
sg_buffer_desc buf_desc;
sg_init_buffer_desc(&buf_desc);

/* ...simply do this: */
sg_buffer_desc buf_desc = { 0 };
```

But that's not all, using zero-initialized fields as 'default' means that
C99 designated initializers can be used, enabling an 'option bag'-style initialization
popular in more dynamic languages like Javascript or Typescript:

```c
/* instead of this: */
sg_buffer_desc buf_desc;
sg_init_buffer_desc(&buf_desc);
buf_desc.type = SG_BUFFERTYPE_INDEXBUFFER;
buf_desc.size = sizeof(indices);
buf_desc.data_ptr = indices;
buf_desc.data_size = sizeof(indices);

/* ...you can now do this: */
sg_buffer_desc buf_desc = {
    .type = SG_BUFFERTYPE_INDEXBUFFER,
    .size = sizeof(indices),
    .data_ptr = indices,
    .data_size = sizeof(indices)
};
```

Missing fields in such a designated initializer block will be set to 
0 (that's why treating the value 0 as default is so important).

It's also possible to put the entire struct initialization into the
resource creation function call:

```c
sg_buffer buf = sg_make_buffer(&(sg_buffer_desc){
    .type = SG_BUFFERTYPE_INDEXBUFFER,
    .size = sizeof(indices),
    .data_ptr = indices,
    .data_size = sizeof(indices)
});
```
This basically gives you optional, named parameters in C. Pretty cool huh?
Nearly feels like a whole new language :)

If used extensively the initialization code can look very 'declarative'.
Whether this is always a good idea remains to be seen, but the nice
thing is that nothing in the sokol_gfx API itself 'enforces' this style.

For instance here is an example from the 
[triangle sample](https://github.com/floooh/sokol-samples/blob/master/glfw/triangle-glfw.c)
which sets up a complete draw-state from a single nested struct
initialization (note the 3 embedded calls to resource creation functions,
these are a bit easy to overlook but important to understand what's going
on):

```c
sg_draw_state draw_state = {
    .pipeline = sg_make_pipeline(&(sg_pipeline_desc){
        .vertex_layouts[0] = {
            .stride = 28,
            .attrs = {
                [0] = { .name="position", .offset=0, .format=SG_VERTEXFORMAT_FLOAT3 },
                [1] = { .name="color0", .offset=12, .format=SG_VERTEXFORMAT_FLOAT4 }
            }
        },
        .shader = sg_make_shader(&(sg_shader_desc){
            .vs.source = 
                "#version 330\n"
                "in vec4 position;\n"
                "in vec4 color0;\n"
                "out vec4 color;\n"
                "void main() {\n"
                "  gl_Position = position;\n"
                "  color = color0;\n"
                "}\n",
            .fs.source =
                "#version 330\n"
                "in vec4 color;\n"
                "out vec4 frag_color;\n"
                "void main() {\n"
                "  frag_color = color;\n"
                "}\n"
        })
    }),
    .vertex_buffers[0] = sg_make_buffer(&(sg_buffer_desc){
        .size = sizeof(vertices),
        .data_ptr = vertices, 
        .data_size = sizeof(vertices)
    })
};
```
Again, you *don't have to* use this style, it's just a C99 language feature,
and (apart from treating the zero-init state as default), isn't "encoded" into
the sokol_gfx.h API.

In C++ this is not yet possible across all compilers (a slightly more
restrictive version of C99 designated initializers are on the roadmap for C++20),
so I may add back a handful of helper functions for the vertex- and
uniform-declarations (which are a bit inconvenient to setup without designated
initializers).
