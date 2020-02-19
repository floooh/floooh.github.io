---
layout: post
title: "sokol_gfx.h Backend Tour: Metal"
---

This is part 3 (and so far, last part) of the blog post mini-series which
looks under the hood of the [sokol_gfx.h](https://github.com/floooh/sokol/blob/master/sokol_gfx.h)
backends. [Part 1](https://floooh.github.io/2020/02/17/sokol-gfx-backend-tour-gl.html)
was mainly about the OpenGL backend, [Part 2](https://floooh.github.io/2020/02/18/sokol-gfx-backend-tour-d3d11.html) about the D3D11 backend, and now it's about the Metal backend.

The Metal backend is on one hand quite different from the other backends
(mainly because it is the only backend written in a different language:
Objective-C, instead of plain C), but it's also very similar to the D3D11
API (mainly because - at least if you squint a bit and ignore the differences between
Objective-C and C - "version 1" of Metal looks a lot like a carefully modernized
and cleaned up version of D3D11, basically what D3D12 could have been if
Mantle wouldn't have happend).

The sokol-gfx Metal backend has a few specialities not found in the other backend:

## Objective-C, ARC and object lifetimes

The Metal backend is written in Objective-C (not Obj-C++), and expects ARC
to be enabled (ARC = Automatic Reference Counting). At the time I originally
wrote the backend this posed a problem: Objective-C object Ids couldn't be
embedded into C structs when compiled as Objective-C because the compiler
couldn't track the lifetime of such Ids embedded into structs (this only
worked when using Objective-C++). In the meantime this seems to have been
fixed, but I think the solution I came up with is better anyway because
it cleanly separates Objective-C "smart data" from C "dumb data".

The Metal backend stores all Objective-C object references in a single
"Id pool" (a simple NSMutableArray holding Ids). Whenever an Objective-C 
object is created, it is stored in that array, and the slot index where
it is stored is returned. This index is then stored in C structs and
used as "object handle" instead of the actual Objective-C Id. Only when
methods must be invoked on the Objective-C object, its reference is
'looked up' in the global 'reference pool'.

This reference pool is also used to control the lifetime of the Objective-C
object. As long as the object is registered in the pool it will be pinned into
memory from ARC's point of view. When the Metal backend no longer needs an
Objective-C, it will be 'released'. This doesn't immediately destroy the object
(since it might still be 'in flight' for rendering), instead it will be added
to a 'deferred-release queue' which will delete the objects when its safe to do so.

## The "Tick Tock" frame-sync model

The entire Metal backend follows a simple 'tick-tock' frame-synchronization
model. While one set of resources is 'in flight' and used to render the current
frame, another set of resources is updated (and rendering commands are
recorded) for the next frame. There is a single sync-point at the end of the
frame (in sg_commit()) where the CPU side needs to check (and worst case: wait)
until the frame before the current 'in-flight' frame has finished. The tick-tock
frame model also provides a simple way to decide when it is safe to delete a resource:
3 frames into the future.


## The Sampler Cache

Metal doesn't have a convenient built-in texture-sampler cache like D3D11 (which returns
an identical sampler object for identical creation parameters), so I implemented the
a similar sampler-cache in the Metal API. 


...TO BE CONTINUED