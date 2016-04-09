---
layout: post
title: A Tour of 3rd Party Code in Oryol (2016)
tags: [oryol]
---

**TL;DR**: a quick run through all the 3rd-party-code that is used in
Oryol as of April 2016, and why it was chosen.

### Small Things

These are mostly header/source pairs which are dropped directly into
the source directory nearby the Oryol code that uses them.

#### ConvertUTF 

This is a simple and very small C header/source pair from the LLVM project which
converts between UTF-8, UTF-16 and UTF-32 and has no further dependencies.

Oryol's convention for string data is to use UTF-8 for internal string
representation, and only convert to 16- or 32-bit 'wide-chars' when talking to
external APIs that don't accept UTF-8.

ConvertUTF is all that's needed to handle all types of international text I've
encountered so far (including Arabian, Korean and various Japanese and Chinese
text representations). No code-page bullshit or global UNICODE defines like it
was hip in Windows a while ago. The use of wchar\_t strings is limited to very
few places, wchar\_t is 16-bits on Windows and 32-bits on UNIX-y platforms, so
it is not a useful general representation for text data.

In the past I called the MultiByteToWideChar/WideCharToMultiByte functions on
Windows and used the iconv() function on UNIX-like systems to convert
between encodings, don't run into the same trap.

- **Source:** [ConvertUTF.h](http://opensource.apple.com//source/lldb/lldb-69/llvm/tools/clang/include/clang/Basic/ConvertUTF.h), [ConvertUTF.c](http://opensource.apple.com//source/lldb/lldb-76/llvm/tools/clang/lib/Basic/ConvertUTF.c)
- **Used in:** Core (StringConverter class)
- **Platforms:** all

#### Whereami

This is also a simple C header/source pair. It's sole purpose is to provide an
absolute path to the own executable on various platforms. This is the most
useful way for cross-platform code to find the installation directory of an
application (as opposed to hacks like writing to the Windows registry at
installation time).

In Oryol this is only used on platforms where the concept of a local filesystem
exists.

- **Source:** [https://github.com/gpakosz/whereami](https://github.com/gpakosz/whereami)
- **Used in:** LocalFS (provides access to local disc filesystem)
- **Platforms:** iOS, Android, Windows, OSX, Linux

#### FlextGL

FlextGL is essentially a python script which generates a single C header/source
pair from the downloaded Khronos XML specs for GL. You provide a small
text file describing what GL version, profile and extensions are required,
and FlextGL generates C glue with just the required code. Before 
FlextGL I used GLEW but found this way too bloated.

- **Source:** [https://github.com/ginkgo/flextGL](https://github.com/ginkgo/flextGL)
- **Used in:** Gfx (only in the desktop GL backends)
- **Platforms:** Windows, Linux, OSX

### External Dependencies

These are pulled in by fips as external dependencies and live outside the
Oryol source tree. Some of them are compiled into static link libs 
during the Oryol build and some are header-only.

#### GLM

GLM is used for all general CPU vector math code in Oryol. Despite
it's name 'OpenGL Mathematics' it is actually a generic math library. 
The name GL only comes in because it tries to mimic the look of GLSL shader
code. 

GLM basically ended my 15-year quest for a usable vector math library that can
be used well in general code (as opposed to ultra-tight inner loops
where it probably isn't the perfect solution - no math lib is I think).

Before GLM I wrote my own vector/matrix/quaternion classes, sometimes wrapping
external libs like XNA Math (yuck).

- **Source:** [https://github.com/g-truc/glm](https://github.com/g-truc/glm)
- **Fips Wrapper:** [https://github.com/floooh/fips-glm](https://github.com/floooh/fips-glm)
- **Used in**: all over the place
- **Platforms**: all

#### GLIML

...as in "GL image loader". This is my own little header-only library to parse
texture data from various compressed-texture file format (DDS, PVR, KTX). It
was my first attempt at a header-only library, so there are some quirks in it
(I should rewrite it in pure C and use STB header conventions). It doesn't have
any allocations, IO or 3D-API calls in it. You just give it a pointer to
texture-file-data in memory, and it fills a simple C++ object with information
that can be directly dropped into GL texture creation functions (but Oryol also
uses GLIML for all other 3D APIs).

I was considering [GLI](http://gli.g-truc.net/0.8.1/index.html) for texture 
loading, but that was a bit overkill since it does a lot more than just
loading textures.

- **Source:** [https://github.com/floooh/gliml](https://github.com/floooh/gliml)
- **Used in:** Assets (TextureLoader class)
- **Platforms:** all

#### GLFW

GLFW is used as platform abstraction wrapper for window and 3D context
creation, and for getting input on desktop platforms when the GL rendering
backend is used. It will also be used in the upcoming Vulkan rendering backend.

Even on some non-GL desktop targets, bits and pieces from GLFW have been ripped
out to get somewhat uniform window- and input-handling across platforms and 3D
rendering backends.

The Windows, OSX and Linux window system APIs, and their ugly cousins, the
glue that connects 3D APIs to those window systems are among the worst
API-contraptions mankind ever came up with, that the GLFW authors are taking
one for the team and wrestle with this mess can't be praised high enough.

If you're still using GLUT, definitely give GLFW a shot.

- **Source:** [https://github.com/glfw/glfw](https://github.com/glfw/glfw)
- **Fips Wrapper:** [https://github.com/floooh/fips-glfw](https://github.com/floooh/fips-glfw)
- **Used in:** Gfx, Input
- **Platforms:** OSX, Linux, Windows when GL rendering backend is used

#### Dear Imgui

ImGui is the first-ever UI framework that I used in my entire life that isn't a
royal pain in the ass to write code for. In fact it is highly enjoyable to use.
If you're grumpy and in a bad mood, just write a few lines of ImGui and you'll
be smiling for the rest of the day. That's how good it is.

And this includes all types of UI frameworks, not just for games.

I would already now use ImGui for everything tools-related. Qt, WPF, WxWidgets
and all that overengineered crap be damned. Just whip up a GL window and do all
that stuff with a 10th of the code, no mind-boggling event handling crap and
butter smooth rendering performance.

The only thing that's missing for all types of UIs is skinnability. It would
be outside the scope of ImGui (one of it's nicest features is it's narrow focus),
but there *should* be a separate, skinnable game UI system that just works like
ImGui. 

- **Source:** [https://github.com/ocornut/imgui](https://github.com/ocornut/imgui)
- **Fips Wrapper:** [https://github.com/fungos/fips-imgui](https://github.com/fungos/fips-imgui)
- **Used in:** IMUI module (just the minimal Oryol rendering wrapper)
- **Platforms:** all

#### SoLoud

SoLoud is a medium-level cross-platform audio lib that's very well suited for
projects that need to keep an eye on code- and asset-sizes. It fits somewhere
between system-level sound APIs like OpenAL and high-level artist-centric audio
solutions like FMOD. It has a very nicely designed simple and intuitive API
which makes simple things simple but complex things possible (yes, really!). Most
medium-level features (like playback or streaming of wav files, OGG support,
various audio filters, SID, TED, MOD playback or speech generation) are
optional in the sense that they don't bloat your exe size if you don't use them.

The special thing about SoLoud integration into Oryol is that it is not
an 'integration' at all. There isn't any glue code which wraps SoLoud into
an Oryol module. Instead you just include the needed SoLoud headers and
start using it. There is no need to simplify the API or pick a subset of
features to make it 'compatible' with Oryol's design philosophy and style, it
just fits in.

- **Source:** [https://github.com/jarikomppa/soloud](https://github.com/jarikomppa/soloud)
- **Fips Wrapper:** [https://github.com/floooh/fips-soloud](https://github.com/floooh/fips-soloud)
- **Used in:** some samples, there is no Oryol wrapper module
- **Platforms:** all except PNaCl (not sure yet whether I'll write a backend
for PNaCl)

#### Remotery

Remotery is a runtime profiler in a single C header/source pair with realtime
visualization running in a web browser. The code areas to be profiled 
are annotated with macros, and the Remotery runtime will measure the time
between profiling events, aggregates them and sends them out over a WebSocket
to the visualization web browser.

- **Source:** [https://github.com/Celtoys/Remotery/](https://github.com/Celtoys/Remotery/)
- **Fips Wrapper:** [https://github.com/floooh/fips-remotery](https://github.com/floooh/fips-remotery)
- **Used in:** all over the place
- **Platform:** all native platforms (iOS, Android, Windows, Linux, OSX)

#### TurboBadger UI

I once integrated TurboBadger UI as a 'game UI' alternative to Dear ImGui
(which is used for debug and tools UIs). Turbobadger is very powerful, it is
skinnable, it has an 'asset file format' and all sorts of bells and whistles.
But it is also a very heavy, old-fashioned UI: it has the traditional 'big
object tree', thousands of refcounted small C++ objects living in a tree, lots
of subclasses with lots of virtual methods, multiple inheritance (yuck) and so
on. As such it doesn't fit well into the Oryol design philosophy.

Of course it is much slimmer than 'professional' (ha..ha..) solutions like
Scaleform, and I would prefer it over most other UI frameworks currently used
in games (including those I wrote myself over the years), but it doesn't have
the immediately obvious brilliance of immediate UIs.

I will most likely move Turbobadger out of core Oryol into an optional addon
module one day.

- **Source:** [https://github.com/fruxo/turbobadger](https://github.com/fruxo/turbobadger)
- **Fips Wrapper:** [https://github.com/fungos/fips-turbobadger](https://github.com/fungos/fips-turbobadger)
- **Used in:** TBUI module
- **Platforms:** all

#### UnitTest++

This is the test framework used in Oryol. There is nothing really special
or fancy about UnitTest++, it just does it's job. I guess that's a good
thing to say about a test framework. I would probably prefer it to be
header-only so that integration into a project would be easier, so I still 
keep looking.

- **Source:** I haven't found an official source repo on github, so I just
dropped everything into the fips-wrapper directory.
- **Fips Wrapper:** [https://github.com/floooh/fips-unittestpp ](https://github.com/floooh/fips-unittestpp)
- **Used in:** most Oryol modules
- **Platform:** all native platforms (I also got it running on emscripten, but
it's a bit hacky and also redundant)

#### libcurl

Curl is used to connect to HTTP servers on some platforms where HTTP support
isn't easily provided by the OS. It is very special compared to the other
dependencies in that it isn't compiled with the rest of the project, but
instead it has been precompiled into static link libs. The reason is the
terribly complex autoconf-based build system setup used by Curl at the time I
wrote the Oryol HTTP module.

There is now official cmake support in curl which might make things easier
(especially for Android), however even if this is the case I'm still looking
for alternatives to curl, since it does much more than what Oryol actually
needs (which is: HTTP, and in the future probably HTTPS and HTTP2).

I would love to see an stb-header-style lib that implements a fully conformant
HTTP (and HTTP2) client, but this is no trivial task.

- **Source:** [https://github.com/curl/curl/](https://github.com/curl/curl/)
- **Fips Wrapper:** [https://github.com/floooh/fips-libcurl](https://github.com/floooh/fips-libcurl)
- **Used in:** HTTP module
- **Platforms:** OSX, Linux, Android

### Other Great Stuff (TM)

#### The STB Headers

I'm a bit surprised that I don't use any of the STB headers directly
in Oryol, but I use them in other projects, and they are included in some
of the 3rd party code listed here. 

In the unlikely case that you never heard of them, the STB headers are
basically the Platonic Ideal of what each library should try to
achieve: they are header-only, written in plain C, don't depend on external
code, are fully configurable and have a liberal open-source license. All of
this together makes them trivial to integrate into C++ projects, which can't be
said of most C++ libs unfortunately.

So far I have mainly used the STB image headers to wipe any dependencies to
libjpeg, libpng (ugh) and DevIL, literally replacing hundreds of source files
and a handful of complex build system setups with 3 simple headers
(stb\_image.h, stb\_image\_resize.h and stb\_image\_write.h).

There's a lot more useful stuff, have a look here: 
[https://github.com/nothings/stb](https://github.com/nothings/stb)

#### JSMN (JSON parser)

If I ever have to parse a JSON file, I'll most likely use
[JSMN](https://github.com/zserge/jsmn). In the past I have used cJSON, but this
has (like many other JSON libs) the disadvantage that it performs (at least)
one allocation for each JSON node. cJSON *does* have the advantage that it
allows to write JSON files with simple code though.

#### enkiTS (threaded task scheduler)

So far I haven't needed a true task system in Oryol yet, but when the time
comes I'll very likely first look into
[enkiTS](https://github.com/dougbinks/enkiTS). A general problem for Oryol is
that emscripten doesn't support traditional threads (well it does now, but this
requires SharedArrayBuffers in browsers, not sure how wide-spread this is yet).

#### Recast (navigation/pathfinding)

As pathfinding solution I would first consider [Recast](https://github.com/recastnavigation/recastnavigation). I already started to play around with it and wrote a fips
wrapper but didn't pursue this further yet since I didn't yet have an idea for
a cool Oryol sample with pathfinding.

#### Bullet (physics/collision)

For physics simulation I would start with
[Bullet](https://github.com/bulletphysics/bullet3) since I know that it works
well with emscripten. Other then that I haven't played around with it yet (I
only have extensive experience with ODE). 

### The End

And that's it. Hopefully I will learn of many other great open source libraries
in the future. In my experience, choosing the right middleware (and deciding
what to tackle myself, and where to use other people's code) has a huge effect
on 'programmer happiness'. 

In the worst case, wrestling with bad middleware can make you hate your job and
there's already enough of this crap in 'professional' game development where
middleware decisions are often not made for purely technical or even rational
reasons. Don't let this happen to you :)


