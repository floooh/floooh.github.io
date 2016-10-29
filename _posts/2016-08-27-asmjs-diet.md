---
layout: post
title: 10 simple diet tricks for asm.js
---

**TL;DR**: reducing the size of asm.js-, and C++-builds in general

Apologies for the click-baity title, I couldn't resist :D

Emscripten demos are sometimes critized for their size and how long it takes to
download and start them. For 'first attempts' this is often a simple oversight,
like uploading an unoptimized version with debug information, or sometimes the
web server's configuration is broken and serves uncompressed Javascript files.

But for more professional demos the simple and inconvenient truth is often that
it is the underlying C++ code which already produces oversized native
executables, and in this case emscripten can't magically produce a small asm.js
file from a bloated code base, the C++ code needs to be trimmed down to
produce a program that's acceptable as webpage content.

This blog post is mainly about 2 things:

1. generally reducing code size in C++ projects
2. adding emscripten-specific tweaks to reduce size further

But first, **what to expect**: as an example for some not-completely-trivial
code here's [my KC85 emulator](http://floooh.github.com/virtualkc), this is
built on top of [Oryol](http://github.com/floooh/oryol), [Dear
Imgui](https://github.com/ocornut/imgui) and the
[SoLoud](https://github.com/jarikomppa/soloud) audio lib, all in all about 110k
lines of C++ code.

The sizes of the native 64-bit version, compiled with -O3 (clang, OSX):

- **1173** KByte uncompressed
- **496** KByte compressed (```gzip --best```)

The same code, compiled with emscripten to asm.js (also with -O3, but the more
detailed compile settings are explained later):

- **2150** KByte uncompressed
- **435** KByte compressed (also ```gzip --best```)

The emscripten version is even smaller than the native version after compression,
impossabru! I actually didn't expect this myself (last time I did such comparisons
the emscripten versions came out about 10..20% bigger), and I have to admit that
I haven't tweaked the native compiler settings as much for output size as the
emscripten settings, so there's possibly room for improvement in the native version.

The C++ code is essentially the same for both versions, only slight differences
when interfacing OSX- vs HTML5-APIs.

From my experience, the simplest asm.js Hello World is around 16 KByte, and
WebGL demos (using Oryol's Gfx module sitting on top emscripten's GL-to-WebGL
translation layer) start at around 100 KByte, and from there on they grow
linearly as features are added, and it doesn't even matter that much whether
the features are written in C/C++, or manually in Javascript. Manually written
Javascript only has a real size advantage if it mainly manipulates the DOM,
because then all the interesting stuff happens in the browser. For lower level
Javascript code that does more than just calling HTML5 APIs, the size
differences to emscripten-compiled code are surprisingly small.

The main takeaway is that **compressed asm.js code can be as small as
compressed native code** and can even rival handwritten Javascript code in some
cases. The actual problem is usually that it is already the native executable
that's bloated, and such bloat is unfortunately quite easy to achieve with C++
(much harder with pure C).

### General C/C++ Tips

#### Dead Code Removal

The most important advice is to write code that makes it easy for the compiler
to **identify and remove dead code**. A program that only uses a very
small feature set of a game-engine should be much smaller than a program that
needs most of the engine's features. Otherwise you'll end up with 
a Tetris clone that's just as big as a full-blown 1st-person-shooter.

In the old times, dead code removal was fairly crude and only happened on the
compilation unit level ('object files', .a/.obj extension). Static libraries
are simple collections of such object files and are treated as such by the
linker. If some other code references at least one symbol in an object file,
the whole file content will become part of the final executable, otherwise the
object file content will be ignored (so the linker doesn't simply add the
whole lib code, but only the contained .obj files that are actually needed).

Some time later 'function level linking' was introduced, instead of looking
at whole object files, the linker would look at single functions to decide
whether they should be part of the build.

Nowadays, compilers can use linktime-code-generation to improve
dead-code-elimination further. Instead of looking at single functions, the
linker looks at the entire merged program code to apply global optimizations
and then removes unreferenced code and data (on the other hand, the compiler
also has more options to inline code, which may increase code size again).

As a simple rule of thumb, a piece of code or data will be removed
during the linker step if nothing points to it. And the easiest
way to achieve this is to make your code as static or as 'non-dynamic'
as possible.

#### Avoid Dynamic Dispatch

For typical C++ code enabling dead code removal mostly boils down to **avoiding
virtual methods and jump tables**, at least on a large scale.

As soon as the compiler needs to store the address of a function somewhere, it
needs to keep the function body in the executable, even if it is never called
(in some cases the compiler might be able to prove that a virtual method is
never called and can still remove it, but never rely on a 'clever compiler'). 

So if your library design relies heavily on subclassing and virtual methods
you're making the life for the compiler much harder.

#### Avoid 'Dynamic Object Factories'

Ok, I don't know what's the proper name for this, but it has bitten me
in the back in past engine architectures: I often had a general object-level
persistency mechanism where objects were saved as a class-id (FourCC or 
string), and a bunch of key-value pairs.

For this to work, there was a general object-factory-mechanism which was able
to construct C++ objects from the class-id. And instead of only registering a
very small set of classes with this object factory (only those that actually
needed persistency), all engine classes registered themselves automatically via
some header-file-macro magic where I had to essentially trick the compiler into
believing a class implementation is needed even though it is never called
directly (only indirectly through base class virtual functions).

The result was that a project using such an engine architecture cannot really
reduce the resulting binary size by removing uneeded features. Even if a game
didn't use particle systems at all, the particle system code was still in the
generated executable.

This sort of magic was (and still is) actually quite common in game engines,
I've seen it quite a bit over the years. But in hindsight this is a terrible
design because it disables any sort of dead-code-removal completely and a lot
of other optimization opportunities with it.

#### Careful with Templates and Inline methods

Templates can actually be useful for code size since they increase the amount
of static type information for the compiler at build time, and may reduce
'dynamicisms', but they can also create bloat because template code is either
inlined, or duplicated for each template specialization.

As a general advice for template code, try to avoid combinatorial 
explosions of template parameters and keep template method bodies 
small, but also don't try to avoid templates at all costs. Small
templated functions will be inlined, and there's not much difference
for the resulting code size whether one or another template specialization
is inlined.

Inline functions increase code size but are important for performance, from my
experience inlining is even more important for performance in asm.js code
compared to native code. In many cases I'm accepting an increase in code size
caused by inlining (within reasonable limits of course). I mostly let the
compiler decide what to inline and what not. When LTO/LTCG is used
(linktime-code-gen), the compiler will also consider 'normal' functions outside
header files for inlining across the entire linked code (not only within the
same compilation unit).

Having said that, I have seen massive performance differences with manual
inline-tweaking in my Z80 CPU emulator, but I think this is a very special
case.

#### Don't use exceptions or RTTI, and disable them when compiling

I'm seeing a 10% size increase when enabling exceptions and RTTI both in the
native and the asm.js executable even though my code doesn't use C++ exceptions
or RTTI at all. So if you can, compile with '-fno-exceptions' and '-fno-rtti'.
This is especially important for emscripten, I don't know all the
emscripten-specific details since I never use C++ exceptions in my own code,
and also avoid 3rd-party-libs that depend on exceptions, but exception handling
in asm.js is more expensive than in native code. May be this will be fixed in
WebAssembly, not that I would care, but I heard some C++ coders actually
do use exceptions ;)

#### Don't use C++ iostreams

Don't use the std::cout stuff, but the old CRT IO functions and printf()
instead.

At least in clang's stdc++ implementation, a lot of code is pulled in for
static constructors to initialize the iostream system even if the actual user
code doesn't use any of it. I'm not sure if this can even be avoided in
native code at all, since linking against the C++ runtime library happens
automatically as soon as C++ is enabled.  In emscripten this was a pretty big
problem which required compiling a manually patched C++ runtime lib, but in the
meantime this has been fixed: emscripten now has two versions of the stdc++
library, one with and one without iostream support, and selects the right one
automatically.

Of course this only works if your code doesn't use the C++ 
iostream stuff, so try to avoid it.

Here's the size difference between a C++-style and a C-style
Hello World in emscripten:

```cpp
// C++ style
#include <iostream>
int main() {
    std::cout << "Hello World!\n";
    return 0;
}
```

This results in **414** KByte uncompressed, and **100** KByte compressed.

And the C-style version, same compiler options:

```cpp
// C style
#include <stdio.h>
int main() {
    printf("Hello World!\n");
    return 0;
}
```

**58** KByte uncompressed, and **17** KByte compressed.

Pretty bad huh? Of course this is a one-time static overhead of about 100
KByte, it doesn't mean that iostream programs are generally 5x bigger, but it's
many little things like this that contribute to overall bloat.


#### Pick your external dependencies carefully

Kinda obvious, but if you depend on external libraries that violate any of the
above, all the effort in your own code is in vain. That's why I prefer
libraries that are written in pure C (like the STB headers), or in a simple
'Orthodox C++' style (like Dear Imgui). For a carefully hand-picked selection
of 3rd-party open source libs that work well with emscripten, see my former
blog post: [A Tour of 3rd Party Code in Oryol
(2016)](http://floooh.github.io/2016/04/09/oryol-3rd-party-code.html).

### Emscripten-specific tuning tips

Emscripten has improved **a lot** for automatically reducing generated code
size since its beginnings, and it has gained a lot of manual tweaking options
to allow even more radical size reductions. The effect of these manual tweaking
options can vary quite drastically between projects, so it's important to play
around with different combinations.

Btw: All the options below and more are described in the file
**src/settings.js** in the emscripten SDK!

From my experience, if the code is already written with reducing code
bloat in mind, the manual tweaking options don't make a big difference
(on my code usually just single-digit KByte numbers), but traditional
C++ code with heavy template use may benefit much more.

Here's a selection of emscripten linker-stage command line args I found useful:

- ```--memory-init-file```: This selects whether static data is embedded as ASCII
  in the asm.js file, or whether a separate binary '.mem' file is generated.
  The original intention was that the binary file is the better option since
  the compressed end result was slightly smaller with the '.mem' file. But
  there's a gotcha that's often overlooked. Some web server configurations
  (e.g. github pages) will only compress a small selection of known file types,
  for instance .js and .txt files are compressed, but not .mem files. And in
  the case of github-pages, and other public hosting solutions, there's no way
  to fix the web server configuration. The unfortunate result is that the final
  download of a .js+.mem file is much bigger because the .mem file is not
  compressed.

- ```-s NO\_FILESYSTEM=1```: This removes the emulated filesystem layer in
  emscripten even if the code contains (but doesn't call) CRT IO functions
  (like fopen/fclose).  This is useful because third party code sometimes has
  such functions even if they are never called. This saves about 70 KBytes
  uncompressed, or 20 KByte compressed (again, not much on its own, but stuff
  like this adds up). 

- ```-s DISABLE_EXCEPTION_CATCHING=1```: this is now actually enabled by default
  at -O1 and above (at least the emscripten docs say so), this will generate
  much simpler exception 'handling' code which simply terminates the program
  instead of calling an exception handler.

- ```--closure 1```: This invokes the Google closure compiler to optimize the
  manually written HTML5 interface code in an asm.js executable. I'm not
  using this anymore since emscripten now has its own minifier when
  closure is not invoked, and in my case it didn't make much of
  a difference. YMMV.

- ```-s ELIMINATE\_DUPLICATE\_FUNCTIONS=1```: this finds and removes
  duplicate function bodies on the asm.js level, which might be useful
  for heavy template use if the code for different template
  arguments ends up being identical. It doesn't make much difference
  on my own stuff (only 2 or 3 KByte in the biggest Oryol samples), but
  this may be useful in different code bases.

- ```--llvm-lto N```: this controls LLVM's LTO (Link-Time-Optimization) pass,
  I have this set to 1 which seems to produce the best tradeoff
  between size and speed for my code. Bigger values seem to do more
  inlining and produce bigger code for little performance increase.

And that's it. There are a lot more detail settings in emscripten's settings.js,
but I think these are the most important for getting the right balance
between code size and performance.

