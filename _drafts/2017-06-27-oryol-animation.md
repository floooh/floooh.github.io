---
layout: post
title: "Oryol: Animation System"
---

A quick walkthrough of the 'small, sloppy & web-friendly' animation module 
I've been for Oryol I've been tinkering on recently.

**Small** means it's just 1.3kloc of code, so it doesn't bloat the
download size too much. **Sloppy** means that I favour performance
over correctness, for instance bone rotation interpolations are done
through simple linear interpolation of the 4 quaternion components, 
not spherical interpolation. As long as the interpolated keys are
fairly nearby this doesn't make a difference, and even big distances
look acceptable to me (eagle-eyed character artists might object).

**Web friendly** means: 

- **no threading**: without shared memory threading, most time would
probably be spent copying data around
- **no SIMD**: this is planned for a later version of WebAssembly, and
in asm.js it will probably never be widely supported
- **no dynamic allocations**: apart from a few handful allocations
during setup, the system runs allocations-free
- **simple rendering**: the animation data is provided in a way that
allows to render skinned characters with as few WebGL calls as
possible (many characters with unique animations can be rendered
in a single draw call)

There are 2 new Oryol samples, one feature demo, and one
benchmark/stresstest demo, available in asm.js and WebAssembly:

Feature Demo **OrbViewer**:

* [asm.js version](http://floooh.github.io/oryol-samples/asmjs/OrbViewer.html)
* [wasm version](http://floooh.github.io/oryol-samples/wasm/OrbViewer.html)

Stresstest **Dragons**:

* [asm.js version](http://floooh.github.io/oryol-samples/asmjs/Dragons.html)
* [wasm version](http://floooh.github.io/oryol-samples/wasm/Dragons.html)

The demos are currently triggering a WebGL crash bug in Firefox on Windows,
this is tracked [here](https://bugzilla.mozilla.org/show_bug.cgi?id=1376399). Since the problem is easy to reproduce, it will hopefully be fixed soon.

### Basic Design

The animation system is an "Oryol Extension Module" and lives in its
own git repo here: [https://github.com/floooh/oryol-animation](https://github.com/floooh/oryol-animation).

I tried to implement the module with a data-oriented approach, related
data is arranged in the order it is consumed by the CPU. I haven't done
serious performance analysis session yet though, so the code will
likely see more tweaks.

Most objects are simple POD (plain-old-data) structs, living in
fixed-capacity linear arrays instead of an dynamic 'object spider-web' held
together by pointers. Instead of pointers, whole groups of objects are
referenced through 'array slices' (aka ranges, aka views). A slice is an
array-like object which doesn't own memory, instead it points into another
array, or even to a random chunk of memory. Since the slices don't own their
memory they must not outlive the memory they point to, but since the object
pools in the animation system have a fixed capacity and never move around
this is not an issue there.

### Immutable Data Structures

The following data is immutable and shared between all 'character instances':

- **AnimLibrary**: A collection of related animation clips, all clips
in the animation library have the same 'curve layout', and only clips
from the same animation library can be mixed with each other
- **AnimClip**: A collection of related animation curves that together
define an animation like 'run', or 'attack'
- **AnimCurve**: A collection of 1..4 dimensional keys with fixed time intervals,
all curves in a clip have the same number of keys, there's a special
'static curve' type, which has the same key value over the entire
animation length, those curves don't take up any space in the key pool. This is
an important optimization, since nearly all translation and scaling components
in a typical skeleton are not animated, only the rotations.
- **Keys**: animation keys don't exist as their own structure, instead they 
live in a single big pool of 32-bit floats.
- **AnimSkeleton**: This is an optional structure which defines a character
bone hiearchy defined as an array of inverse bind-pose matrices in model
space, and an array of parent indices. They are optional because animation 
sampling and mixing also works without character skeletons.

All those immutable objects live in fixed-capacity arrays which are
allocated when the animation system is initialized (so the application needs
to know how many items of each type can exist at most at the same time).

Here's a drawing how it all fits together:

[FIXME]

### Key Tables

[FIXME]

### AnimInstances

### Sampling and Mixing

### Skeleton Evaluation

### Rendering










