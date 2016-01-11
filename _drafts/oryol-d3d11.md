---
layout: post
title: Oryol's D3D11 Renderer
---

Because 'blog' is short for 'backlog', right? ;)

When I moved my blog from Blogger over to github a few days ago I realized that
I didn't write a single blog post in 2015. The main reason was that I spent
most of my spare time working on github stuff:

Oryol has gained **3 new rendering backends** (D3D11, Metal and D3D12). I was
actually hoping make the bunch complete and also add a Vulkan renderer, but
alas, Vulkan didn't quite make it and was moved to 2016. Instead I spent some
time towards the year's end on a **[KC85
emulator](http://floooh.github.io/virtualkc/)**, an idea that I was carrying
around in my head since 2013.

But back to the topic of this post: Oryol's D3D11 renderer. Originally I saw
the D3D11 backend just as a proof-of-concept for the Oryol Gfx API (until
then, only OpenGL was supported), and as a stepping stone towards D3D12. But
now that it is done, I would recommend it as the main rendering backend 
for Windows. The code is much smaller and cleaner than both the GL
and D3D12 renderer, and being limited by the WebGL feature set, Oryol
cannot really make use of D3D12 (and I seriously doubt that the majority
of real-world games will benefit from D3D12 at all, more on that in a 
later post).

## D3D11 Renderer Overview

Work on the D3D11 renderer started mid of May 2015, and was mostly finished
around mid of June with most actual programming work done on the weekends.

The render backend has about 2.5k lines-of-code (without comments, in contrast,
the GL render backend has 3.1 kloc).

Changes to the Gfx module's API were minimal, granular shader variable 
updates have been replaced with 'uniform block' updates, but that
was about it.

Writing the actual D3D11 code was the easy part, the hard parts that had to
be solved was the loss of GLFW as window-management- and input-wrapper, 
and extending the shader code generator to support HLSL and grouping
shader variables into uniform blocks in a way that's still compatible
with WebGL and GLES2.

## The Easy Parts

Let's quickly skim over the easy parts:

### D3D Device and Swap Chain
(TODO)

### Meshes
(TODO)

### Textures
(TODO)

### Shaders
(TODO)

### Draw States
(TODO)

### Applying State and Drawing
(TODO)

## The Hard Parts

### Window Management and Input
(TODO)

### Shader Code Generation
(TODO)

### Uniform Blocks, Textures and GLES2
(TODO)

