---
layout: post
title: An Accidential Unbloating
---

Here's a riddle: 

Replacing 2.7 kloc C++ code with 3.6 kloc C code that does roughly the same
results in a size reduction of WebAssembly- and x86/64-binaries by
60 to 70 KBytes.

This happened when I replaced the Gfx module's rendering backend in
[Oryol](https://www.github.com/floooh/oryol) with
[sokol_gfx.h](https://www.github.com/floooh/sokol). A detailed rundown
of the numbers is [here](https://docs.google.com/spreadsheets/d/1N4p0njT-6OsbldCE3sNPYk-mdbTVNMDKt60lpf__vs8/edit?usp=sharing).

The massive size reduction (up to 50% of the total size for simple demos) is
both surprising and embarrasing, because I was priding myself that Oryol is
already quite bloat-free (it's even the whole point why Oryol exists).

With 'traditional methods' (described in this blog post: 
["10 simple diet tricks for asm.js"](http://floooh.github.io/2016/08/27/asmjs-diet.html)
I was soon running into diminishing returns, and chopping off another 1 or 2
percent was already a pretty big success.

Seeing a 25% to 50% drop from a change that wasn't even intended to reduce
binary size was a pretty big surprise to put it mildly (I was actually expecting
a slight size increase in the single-digit KByte range because there's an additional
translation happening for structs and enums during resource creation).

Now the question is of course: **why** does replacing 2.7 kloc fairly
average looking C++ code with 3.6 kloc (also fairly average looking) C code
result in dramatically less compiled code?

[Bloaty McBloatface](https://github.com/google/bloaty) to the rescue! This
only works on ELF and Mach-O object files or executables, but since the
size reductions also happen in native executables that's fine.

Among other things, bloaty can create a list of functions, sorted by the
size they take up in an executable. And even more useful, it can create a
diff view between 2 versions of the same executable:

```
     VM SIZE                                                                   FILE SIZE
 --------------                                                             --------------
  [NEW] +1.98Ki _sg_setup                                                   +1.98Ki  [NEW]
  [NEW] +1.11Ki _sg_destroy_all_resources()                                 +1.11Ki  [NEW]
  [NEW]    +912 _sg_begin_pass()                                               +912  [NEW]
  [NEW]    +816 Oryol::_priv::sokolGfxBackend::Setup()                         +816  [NEW]
  [NEW]    +656 _sg_reset_state_cache()                                        +656  [NEW]
  [NEW]    +592 _sg_end_pass                                                   +592  [NEW]
  [NEW]    +552 _sg_gl                                                            0  [ = ]
  [NEW]    +544 _sg_shutdown                                                   +544  [NEW]
  [NEW]    +448 Oryol::ResourceLabelStack::Setup()                             +448  [NEW]
...
  [DEL]    -896 Oryol::_priv::textureBase::Clear()                             -896  [DEL]
  [DEL]    -912 Oryol::_priv::glRenderer::glRenderer()                         -912  [DEL]
  [DEL]    -912 Oryol::_priv::glRenderer::setup()                              -912  [DEL]
 -51.7%    -976 Oryol::Gfx::Setup()                                            -976 -51.7%
  [DEL] -1.02Ki Oryol::Map<>::Erase()                                       -1.02Ki  [DEL]
  [DEL] -1.08Ki Oryol::_priv::glRenderer::beginPass()                       -1.08Ki  [DEL]
  [DEL] -1.17Ki Oryol::_priv::gfxResourceContainer::setup()                 -1.17Ki  [DEL]
  [DEL] -1.17Ki Oryol::_priv::meshBase::Clear()                             -1.17Ki  [DEL]
  [DEL] -1.22Ki Oryol::_priv::gfxResourceContainer::discard()               -1.22Ki  [DEL]
  [DEL] -1.23Ki Oryol::ShaderSetup::programEntry::programEntry()            -1.23Ki  [DEL]
 -46.6% -1.27Ki Oryol::Array<>::Add<>()                                     -1.27Ki -46.6%
  [DEL] -1.36Ki Oryol::_priv::gfxResourceContainer::destroyResource()       -1.36Ki  [DEL]
  [DEL] -1.45Ki Oryol::_priv::gfxResourceContainer::~gfxResourceContainer() -1.45Ki  [DEL]
  [DEL] -1.45Ki Oryol::_priv::pipelineBase::Clear()                         -1.45Ki  [DEL]
  [DEL] -1.50Ki Oryol::ShaderSetup::programEntry::operator=()               -1.50Ki  [DEL]
  [DEL] -1.66Ki Oryol::_priv::glShader::glShader()                          -1.66Ki  [DEL]
  [DEL] -1.72Ki Oryol::StaticArray<>::operator=()                           -1.72Ki  [DEL]
 -63.6% -1.83Ki Oryol::StaticArray<>::StaticArray()                         -1.83Ki -63.6%
  [DEL] -1.89Ki Oryol::Array<>::adjustCapacity()                            -1.89Ki  [DEL]
  -3.0% -1.96Ki [__TEXT,__cstring]                                          -1.96Ki  -3.0%
  [DEL] -2.22Ki Oryol::_priv::shaderBase::Clear()                           -2.22Ki  [DEL]
 -16.5% -2.81Ki [__TEXT]                                                    -2.81Ki -16.5%
  [DEL] -3.41Ki Oryol::_priv::glPipeline::glPipeline()                      -3.41Ki  [DEL]
  [DEL] -4.72Ki Oryol::ResourcePool<>::Setup()                              -4.72Ki  [DEL]
 -12.5% -8.00Ki [__LINKEDIT]                                                -8.00Ki -12.8%
 -17.1% -52.0Ki TOTAL                                                       -52.0Ki -17.7%
```

This is a (shortened) diff-view of the 'Clear' sample, the most simple and smallest
Oryol Gfx module sample).

At the top is the stuff that's new or has increased in size (for instance all the
\_sg\_* entries are the new sokol_gfx.h C functions), and at the bottom
is all the stuff that disappeared or got smaller.

The first thing that's immediately obvious is that there isn't a single cause
for the size reduction. More, bigger functions were replaced with fewer,
smaller functions: only 2 functions above 1 KByte were added, but 18
functions bigger than 1 KByte were removed or got a lot smaller.

The next interesting (and also most important) info that can be gathered from
the function names is that most of the code that disappeared (and thus
caused the code bloat in the old C++ backend) is either about graphics
resource management or template container classes. The actual rendering
code is mostly just short sequences of GL calls and mostly the same
between the C++ and C version, so it makes sense that it doesn't show
up here.

The resource management code does roughly the same in the C++ and C version,
but it's not a 'dumb rewrite' from C++ to C. The C version has fewer, simpler
data structures, and the C++ code contains quite a bit of template container
code for resource object housekeeping, not only for the five toplevel
resource object types but also for smaller embedded items like shader
uniforms or vertex components which were also implemented as C++ classes,
sometimes with non-trivial copy-behaviour. Naturally, in the C version those
embedded items are all just plain-old-data items which can be copied with a
simple memory copy. My educated guess is that simple stuff like this adds up
and is the main reason for the observed size difference.

Another potential reason of why the C++ code could be bigger than similar
C code would be range checks in C++ container classes, but both in 
sokol_gfx as well as in the Oryol container classes all range checks
are removed in optimized builds, so this turned out to be a non-issue 
(whether removing range checks in release mode is a good idea is another topic).

I'll keep it at that for now. The plan was anyway to gradually replace the
platform wrapper parts in Oryol with more flexible C headers (which can be
more easily integrated into existing projects than the Oryol framework) and
only keep a thin C++ 'sugar coating layer' over the new C core. That this
approach also results in smaller binaries wasn't the motivation behind
the effort, but it's certainly a very welcome side effect.

I think the main take-away is: while a carefully crafted C++ codebase should
*not* automatically result in bigger executables compared to implementing
the same functionality in C, it is easier in C++ to accidently introduce code
bloat by 'careless' use of template code, especially containers. Not because
template code is automatically bigger than 'handcrafted code', but because
C++ containers are easy to throw in (instead of stopping and considering
whether such a 'generic solution' is the right way to go). Since C has no
generics and no operator overloading, adding accidential complexity is much
harder.

The situation is a bit like working around the garbage collector in
memory-managed languages: It is possible to write 'garbage free' code in GC
languages, but it requires detailed knowledge of how the specific garbage
collector works, and code to work around the garbage collector often looks
'non-idiomatic' (see asm.js for example). So avoiding garbage collection in a
GC language adds a lot more mental burden, probably more mental burden than
implementing the same stuff in a non-managed language to begin with.
