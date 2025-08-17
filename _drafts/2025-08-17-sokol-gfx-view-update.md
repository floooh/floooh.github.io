---
layout: post
title: The sokol-gfx resource view update.
---

In a couple of days I will merge the next big (and breaking) sokol-gfx
update which adds resource view objects and in turn removes pre-baked
pass-attachment objects.

## What are resource view objects?

If you're familiar with D3D10 and later you'll feel right at home since
resource views are a fundamental concept in D3D, and sokol-gfx's idea
of resource views is closest to D3D11. Other 3D APIs either don't have
view objects at all (WebGL and GL <4.3), or only associate resource
views with texture data (GL >= 4.3, Metal and WebGPU).

In a nutshell, sokol-gfx view objects specialize an image or
buffer resource object for a specific binding type:

- sampling a texture in a shader requires a **texture view**
- accessing a storage image in a compute shader requires a **storage image view**
- accessing a storage buffer in a shader requires a **storage buffer view**
- each render pass attachment type requires its own view object:
    - **color attachment views**
    - **resolve attachment views**
    - **depth-stencil attachment views**

I was actually considering calling this new object type `sg_binding`, but
since 'view' is the more established term I went with `sg_view` instead.

## New 'unlocked' features

This first sokol-gfx resource view update unlocks the following features:

- Storage buffer bindings can now have an offset, previously only vertex-
  and index-buffer bindings could have an offset. Binding storage buffers
  with offsets is mainly useful when the same buffer contains different
  types of items in different sections of the buffer, and processing
  those items in separate compute shaders.
- Texture views can define a subset of the parent image by defining
  their own mipmap- and slice-range.
- Storage images are no longer 'compute pass attachments', but instead
  bound like regular textures in the `sg_apply_bindings()` call. This
  allows to write to many different storage images in the same compute pass
  (the number of simultanously bound storage images is still very restricted
  though)
- Combinations of render pass attachment images are no longer 'pre-baked'
  into `sg_attachments` objects. Instead `sg_attachments` is now a
  transient struct similar to `sg_bindings`. This relaxes another
  'combinatorial explosion' scenario because rendering code longer needs
  to predict all possible render attachment combinations upfront.

## Current restrictions and planned features

The following features are planned for a followup 'resource view update 2':

- Currently it's not possible to re-interpret the pixel format
  or image type in a view object, this is planned for update 2.
- The maximum number of resource bindings of a specific type is currently
  hardwired to very conservative limits (up to 4 storage image bindings,
  up to 8 storage buffer bindings per stage), those very conservative
  hardwired limits will most likely be replaced with dynamic limits
  defined in `sg_limts`.

For more details about planned 'update 2' features see:

[https://github.com/floooh/sokol/issues/1302](https://github.com/floooh/sokol/issues/1302)


## Working with resource view objects

### Shader Authoring Changes

When using sokol-shdc, the only change on the shader side is that textures,
storage buffers and storage images now share a common bindslot range, previously
each binding type had its slot range:

```glsl
@cs cs
layout(binding=0) uniform texture2D cs_inp_tex;
layout(binding=0, rgba8) uniform writeonly image2D cs_outp_tex;
// ...
@end
```

Note how the texture- and storage-image bindings use the same bindslot 0
because previously textures and storage images had their own bindslot space.

This code will now produce a 'bindslot collision error' when compiled with
sokol-shdc, because texture- and storage-image bindings now use the same bindslot
space, so bindings need to be fixed to no longer collide:

```glsl
@cs cs
layout(binding=0) uniform texture2D cs_inp_tex;
layout(binding=1, rgba8) uniform writeonly image2D cs_outp_tex;
// ...
@end
```
This bindslot fixup is the only change required on the shader side.

### Texture Views

Let's say a shader defines a texture binding at slot 3:

```glsl
layout(binding=3) uniform texture2D tex;
```

To 'populate' this bindslot on the CPU side you now need two objects: an image
object, and a texture view on the image object:

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 4,
    .height = 4,
    .data.subimage[0][0] = ...,
});
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture = { .image = img },
})
```

The `sg_apply_bindings()` call now has an array of `sg_view` handles instead
of separate arrays for images and storage buffers:
```c
sg_apply_bindings(&(sg_bindings){
    .vertex_buffers[0] = ...,
    .views[VIEW_tex] = tex_view,
    .samplers[SMP_smp] = ...,
});
```

Since the texture binding was defined as `layout(binding=3)` it's also
safe to just use the bind slot index directly instead of the code-generated
constant:

```c
sg_apply_bindings(&(sg_bindings){
    .vertex_buffers[0] = ...,
    .views[3] = tex_view,
    .samplers[SMP_smp] = ...,
});
```
...using the code-generated constant is just more readable, since the
connection between the `sg_bindings` bindslot and the binding in the
shader source code becomes clearer.

In many situations you only need the view handle and don't need the
separate image handle, this means you can nest the `sg_make_image()`
inside the `sg_make_view()` call:

```c
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture.image = sg_make_image(&(sg_image_view){
        .width = 4,
        .height = 4,
        .data.subimage[0][0] = ...,
    }),
});
```

If you still need the image handle later you can extract it from the
view object via `sg_query_view_image()`:
```c
sg_image img = sg_query_view_image(tex_view);
```

Texture views can select a subrange of mipmaps and slices of their parent
image:
```c
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture = {
        .image = img,
        .mip_levels = { .base = 1, .count = 3 },
        .slices = { .base = 5, .count = 2 },
    },
});
```

If `.count` is left at default-zero it means 'all remaining mipmaps or slices'.
For instance this will only skip the most detailed mipmap but keep the
remaining mipmap chain in place:

```c
sg_view tex_view = sg_make_view(&(sg_view_desc){
    .texture = {
        .image = img,
        .mip_levels = { .base = 1 },
    },
});
```

## View vs parent resource lifetime considerations

Before moving on to the other view types, a little interlude about
lifetimes and resource states:

If you're coming from 3D APIs with ref-counted lifetime management like D3D
Metal you might be tempted to 'release' the image object right after creating
its view object since the image object handle isn't really needed anymore
(but is kept alive by the 3D API because the view bumped the refcount of
the parent resource):

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 4,
    .height = 4,
    .data.subimage[0][0] = ...,
});
sg_view tex_view = sg_make_view(&(sg_view){ .texture.image = img });
sg_destroy_image(img);
```

In sokol-gfx lifetimes are explicit, if you pull the rug under a view
like this nothing catastrophic will happen (e.g. no crashes or hard
validation layers errors), but rendering operations involving such 'incomplete'
views will be silently skipped (this is basically the same behaviour as before
when trying to render with images or buffers in a non-valid resource state).

Another slightly counter-intuitive behaviour might be that a view object
remains in valid state despite its parent resource being destroyed, e.g.
following the above example code:

```c
// get the destroyed image's resource state
if (sg_query_image_state(img) == SG_RESOURCESTATE_INVALID) {
    // if-branch taken, since the image had been destroyed
    ...
}
// get the image's texture view resource state
if (sg_query_view_state(tex_view) == SG_RESOURCESTATE_VALID) {
    // if-branch *also* taken!
}
```

I went a bit back and forth on this decision but I think the behaviour makes
sense from the perspective that all resource state changes in sokol-gfx
are explicit (e.g. there are no 'automatic' state changes as
a side effect of some other state change, all resource state changes
are directly caused by a function call on that resource object. The same
has always been true for pipelines and their shader object, just not
specifically documented.

If you want to check whether a view is 'renderable' you can use
the following shortcut:

```c
if (sg_query_image_state(sg_query_view_image(tex_view)) == SG_RESOURCESTATE_VALID) {
    // the view is 'renderable'
}
```
This works because no matter what state the view object is in (or even exists),
`sq_query_view_image()` will either return an image handle or an invalid handle
and both can be passed into `sg_query_image_state()`. An invalid image handle
will return `SG_RESOURCESTATE_INVALID` while a valid image handle will return
the actual `SG_RESOURCESTATE_*` of the image object.

If the parent resource goes through a 'destroy/make' or 'uninit/init' cycle,
all views which had been created from this parent resource must also be
recreated.

A common pattern for this situation is to use `uninit/init` because the
handles will remain valid (e.g. you don't need to distribute new object
handles into all corners of your code base):

```c
// first unit/init the parent image with new init params:
sg_unit_image(img);
sg_init_image(img, &(sg_image_desc){ ... });
// then 'cycle' the image's view objects
sg_uninit_view(tex_view);
sg_init_view(tex_view, &(sg_view_desc){ .texture.image = img });
```

I was at first considering to add a 'managed mode' for views which would track
the state of their parent resource and automatically go through an uninit/init
cycle when needed, but this just didn't fit sokol philosophy of lifetimes and
resource states being explicit, and having this one special case for view
objects caused more confusion which wasn't worth the small gain in convenience
(this decision also wasn't purely based on gut feeling since I actually *had*
implemented the 'managed mode' already but then kicked it out again after
actually starting to port the sokol sample code over).

When porting existing code over to resource view object, don't forget
that you need to destroy at least two objects now. It's tempting to just
replace all image handles with view handles, and then only destroy the
view but forget to destroy the image. That's a good way to add a quickly
growing memory leak ;) (at some point the image or buffer pools will be
exhausted though and resource creation will start to fail - that's why
it's a good idea to tweak the pool sizes in the `sg_setup()` close to
the maximum number of resource objects of each type you are expecting.


The order in which you destroy the views and parent resources doesn't
matter, this:
```c
sg_destroy_view(view);
sg_destroy_image(img);
```
...works just as well as this:
```c
sg_destroy_image(img);
sg_destroy_view(view);
```

**BE AWARE OF THIS TRAP:**
```c
sg_destroy_view(view);
sg_destroy_image(sg_query_view_image(view));
```

Since the view is already destroyed, `sg_query_view_image()` will return the invalid
handle, and passing the invalid handle into `sg_destroy_image()` is a silent no-op
(e.g. your image will leak).

...this is actually a nice example of how convenience in one situation (calling
`sg_query_view_image(view)` and `sg_destroy_image()` with an invalid handle
being a silent no-op) can cause trouble in other situations. I'll need to think
about whether this should at least be logged as an error instead.