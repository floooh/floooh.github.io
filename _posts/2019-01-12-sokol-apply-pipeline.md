---
layout: post
title: A small sokol_gfx.h API update
---

I just finished a small update for sokol_gfx.h which makes the API a bit
less awkward to use in some situations. Existing code doesn't break,
but some things have been deprecated.

The main change is that pipeline state and resource bindings for the 
next draw call are now set in separate calls:

- **sg_apply_draw_state()** has been replaced with two separate functions:
    - **sg_apply_pipeline()** to set the pipeline state
    - **sg_apply_bindings()** to set the resource bindings
- the struct **sg_draw_state** has been replaced with **sg_bindings**, the new struct only contains the bind slots for buffers and images, but doesn't contain the pipeline state object
- and a purely cosmetic change: the function **sg_apply_uniform_blocks** has been renamed to **sg_apply_uniforms()**

Renaming *draw_state* to something else was something I wanted to do
for a long time, since this name is unusual and doesn't appear 
anywhere in the underlying system 3D-APIs. The main purpose of a *draw_state*
was to define the *resource bindings* for the next draw call (bind-slots to be
filled with buffers and images), ...and it also sets the associated
pipeline state.

Bundling the pipeline state with the resource bindings makes sense in some
situations, but not when the resource bindings are updated with a higher
frequency than the pipeline state. Performance-wise this wasn't much 
of a problem, you simply called sg_apply_draw_state() again with
different buffers and images in the bind-slots, but kept the same
pipeline state object. The 3D-API backends would notice that the
pipeline state doesn't need to change and skip directly to
updating the bind slots.

Doing the pipeline and resource binding update in the same call also
simplified validation a bit, because the resource binding configuration
must match the pipeline state object (e.g. vertex layout, shaders 
expecting images in certain shader stages and image slots).

But there was one situation where the required code was really awkward: when
the resource bindings change between draw calls, but not the shader uniforms.

For instance look at this render-backend loop for Dear ImGui:

```cpp
sg_apply_draw_state(&draw_state);
sg_apply_uniform_block(SG_SHADERSTAGE_VS, 0, &vs_params, sizeof(vs_params));
int base_element = 0;
for (const ImDrawCmd& pcmd : cl->CmdBuffer) {
    if (pcmd.UserCallback) {
        pcmd.UserCallback(cl, &pcmd);
    }
    else {
        if (tex_id != pcmd.TextureId) {
            tex_id = pcmd.TextureId;
            draw_state.fs_images[0].id = (uint32_t)(uintptr_t)tex_id;
            sg_apply_draw_state(&draw_state);
            sg_apply_uniform_block(SG_SHADERSTAGE_VS, 0, &vs_params, sizeof(vs_params));
        }
        const int scissor_x = (int) (pcmd.ClipRect.x);
        const int scissor_y = (int) (pcmd.ClipRect.y);
        const int scissor_w = (int) (pcmd.ClipRect.z - pcmd.ClipRect.x);
        const int scissor_h = (int) (pcmd.ClipRect.w - pcmd.ClipRect.y);
        sg_apply_scissor_rect(scissor_x, scissor_y, scissor_w, scissor_h, true);
        sg_draw(base_element, pcmd.ElemCount, 1);
    }
    base_element += pcmd.ElemCount;
}
```

Especially this part where the texture changes:

```cpp
if (tex_id != pcmd.TextureId) {
    tex_id = pcmd.TextureId;
    draw_state.fs_images[0].id = (uint32_t)(uintptr_t)tex_id;
    sg_apply_draw_state(&draw_state);
    sg_apply_uniform_block(SG_SHADERSTAGE_VS, 0, &vs_params, sizeof(vs_params));
}
```

This puts the new texture id into the first fragment shader image slot and
calls *sg_apply_draw_state()* to update the resource bindings. So far so
good. But notice how I also need to call *sg_apply_uniform_block()* even
though the uniform data doesn't need to change! This is because the call to
*sg_apply_draw_state()* **may** have switched to a different pipeline,
and thus to a different shader, invalidating the old uniform data.

The same code updated for *sg_apply_pipeline()* looks like this:

```cpp
sg_apply_pipeline(pip);
sg_apply_bindings(&bind);
sg_apply_uniforms(SG_SHADERSTAGE_VS, 0, &vs_params, sizeof(vs_params));
int base_element = 0;
for (const ImDrawCmd& pcmd : cl->CmdBuffer) {
    if (pcmd.UserCallback) {
        pcmd.UserCallback(cl, &pcmd);
    }
    else {
        if (tex_id != pcmd.TextureId) {
            tex_id = pcmd.TextureId;
            bind.fs_images[0].id = (uint32_t)(uintptr_t)tex_id;
            sg_apply_bindings(&bind);
        }
        const int scissor_x = (int) (pcmd.ClipRect.x);
        const int scissor_y = (int) (pcmd.ClipRect.y);
        const int scissor_w = (int) (pcmd.ClipRect.z - pcmd.ClipRect.x);
        const int scissor_h = (int) (pcmd.ClipRect.w - pcmd.ClipRect.y);
        sg_apply_scissor_rect(scissor_x, scissor_y, scissor_w, scissor_h, true);
        sg_draw(base_element, pcmd.ElemCount, 1);
    }
    base_element += pcmd.ElemCount;
}
```

Note how the pipeline and uniforms are set only once outside the loop,
and when the texture needs to change, only the resource bindings
need to be updated with a call to *sg_apply_bindings()*, and no
call to sg_apply_uniforms() is needed.

You can check the changes in [this merge commit](https://github.com/floooh/sokol/commit/ea9b836153acc5088ea6b10a8308d02179a7f054).

And the updated [sokol-samples here](https://github.com/floooh/sokol-samples/commit/69f9e713e37d9f33f4b13eeb93f9b01f026b216c).

To help with converting your code, you can define **SOKOL_NO_DEPRECATED** before
including sokol_gfx.h, this will remove the deprecated structs and function names
so that your code fails to compile.

And that is all I think.