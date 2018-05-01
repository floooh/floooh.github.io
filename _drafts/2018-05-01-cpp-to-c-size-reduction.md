---
layout: post
title: An unexpected size reduction
---

TL;DR: replacing 2.7 kloc C++ with 3.6 kloc C reduces size of program
binary by 50..70 KBytes

After spending a bit more time than planned with emulator coding (but hey
there's a 'mostly working' [C64 emulation in YAKC
now](https://floooh.github.com/virtualkc)) I continued work on moving the
[Oryol](https://www.github.com/floooh/oryol) Gfx module on top of
[sokol_gfx.h](https://www.github.com/floooh/sokol). It's mostly finished now,
apart from fixing minor bugs and making sure that everything works on all
platforms. The plan was to write one or two blog posts about API changes, and
then a few weeks later, merge the changes back to the Oryol master branch.

However, something interesting and very unexpected happened, so here's
a different blog post first:

After I fixed all the Oryol samples, I wanted to check how much bigger the
resulting WebAssembly blobs had become. I was expecting that they've grown by
a few KBytes, because there's now an additional translation step happening
for structs and enums when creating graphics resource objects. Previously,
the Gfx module types were directly translated to GL, D3D11 or Metal types
when creating resource objects, but now the Gfx types are first translated to
sokol-gfx types, and then from sokol-gfx types to the native 3D API types.

Apart from that additional translation layer, the actual rendering backend
code is sokol_gfx.h is doing pretty much the same things as the old
Gfx module backends, but sokol_gfx.h is written in C instead of C++.

What I was expecting was a size increase of maybe 2 or 3 KBytes for WASM,
but what I got was a size **decrease** of around 70 KBytes, and this
was pretty much the same for all samples that did 3D rendering. The most
simple demo (which just clears the framebuffer), dropped from about 113 KBytes
to 56 KBytes!

That was a shocking outcome, because I prided myself that Oryol is a fairly
bloat free C++ framework, in a way that's the whole point why Oryol exists,
because I wasn't happy with the resulting download sizes for a lot of
emscripten-compiled C++ code (including my own former attempts).

For Oryol, I've been carefully selecting an architecture, C++ subset and
build options that favours a small download size while still giving good
performance and coding convenience. I've been running into dimishing returns
to a point where I was happy to shave off 2 or 4 KBytes from a 200 KByte binary.

And there I go and rewrite some C++ to C and suddenly I'm seeing the biggest
size reductions since the start of the project!




