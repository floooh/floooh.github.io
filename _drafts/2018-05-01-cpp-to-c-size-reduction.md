---
layout: post
title: An unexpected size reduction
---

Here's an interesting riddle: 

Replacing 2.7 kloc C++ code with 3.6 kloc C code that does roughly the same
reduces the size of WebAssembly and 64-bit x86 binaries by 60 to 70 KBytes in
Oryol, which for the most simple WebGL demo which just clears the canvas means
the WASM size is halved from 113 KBytes down 57 KBytes).

This happened when I replaced the Gfx module's rendering backend in
[Oryol](https://www.github.com/floooh/oryol) with
[sokol_gfx.h](https://www.github.com/floooh/sokol).

The massive size reduction is both surprising and embarrasing, because
I was priding myself that Oryol is quite bloat-free. This is even one of
the main reasons why Oryol was created, instead of trying to port an existing
3D engine to asm.js/wasm, which resulted in binary sizes that were much bigger
than similar 'hand-written' Javascript code.

With 'traditional methods' (detailed in this blog post: 
["10 simple diet tricks for asm.js"](http://floooh.github.io/2016/08/27/asmjs-diet.html)
I was soon running into diminishing returns, and chopping off another 1 or 2
percent was already a pretty big success.

So seeing a 25% to 50% drop from a change that wasn't even intended to reduce
binary size was a pretty big surprise to put it mildly (I was actually expecting
a slight size increase in the 2..3 KByte range because there's an additional
translation happening for structs and enums during resource creation).

Now the question is of course: **why** does replacing 2.7 kloc fairly
average looking C++ code with 3.6 kloc (also fairly average looking) C code
result in dramatically less compiled code?

[Bloaty McBloatface](https://github.com/google/bloaty) to the rescue! This
only works on ELF and Mach-O object files or executables, but that's fine
since the size reduction also carries over to x86/64 code (I've created
a little Google Docs spreadsheet with before/after numbers for all
Oryol samples [here](https://docs.google.com/spreadsheets/d/1N4p0njT-6OsbldCE3sNPYk-mdbTVNMDKt60lpf__vs8/edit?usp=sharing)).

Among other things, bloaty can create a list of functions, sorted by the
size they take up in an executable, and even more useful, it can create a
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
  [NEW]    +432 Oryol::_priv::sokolGfxBackend::~sokolGfxBackend()              +432  [NEW]
  +119%    +400 Oryol::_priv::elementBuffer<>::alloc()                         +400  +119%
  [NEW]    +304 _sg_begin_pass                                                 +304  [NEW]
  [NEW]    +272 Oryol::_priv::sokolGfxBackend::BeginPass()                     +272  [NEW]
  [NEW]    +256 _sg_begin_default_pass                                         +256  [NEW]
  [NEW]    +224 Oryol::Array<>::Add()                                          +224  [NEW]
  [NEW]    +216 _sg                                                               0  [ = ]
  [NEW]    +112 Oryol::StringAtom::setupFromCString()                          +112  [NEW]
  [NEW]    +112 Oryol::_priv::sokolGfxBackend::Discard()                       +112  [NEW]
  [ = ]       0 __glfw                                                         +112   +30%
   +62%     +80 ClearApp::OnInit()                                              +80   +62%
...
  [DEL]    -672 Oryol::ResourceRegistry::Remove()                              -672  [DEL]
  [DEL]    -704 Oryol::_priv::glRenderPass::glRenderPass()                     -704  [DEL]
 -85.7%    -768 Oryol::_priv::elementBuffer<>::erase()                         -768 -85.7%
  [DEL]    -832 Oryol::Queue<>::checkEnqueue()                                 -832  [DEL]
  [DEL]    -848 Oryol::_priv::glCaps::printInfo()                              -848  [DEL]
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

This is a (shortened) diff-view of the Clear sample (the most simple, and smallest
Oryol sample which uses the Gfx module, where the WASM blob size was cut in half).

At the top is the stuff that is new or increased in size (for instance all the
\_sg\_* functions are the new sokol_gfx.h C functions), and at the bottom
is all the stuff that disappeared or got smaller.

The first thing that's immediately obvious is that there isn't a single cause
for the size reduction. Instead the average size of removed functions was
bigger than the average size of new functions (only 2 functions above 1 KByte
were added, but 18 functions bigger than 1 KByte disappeared).

The next interesting (and most like also most important) info is that most of
the functions that disappeared (and thus caused the code bloat in the old C++
backend) are either about graphics resource management or template container
classes. The two are related, because the graphics resource structs in the
old C++ code where kept in container classes (like Oryol::Array<> and
Oryol::Map<>), and also sometimes contained embedded arrays like
Oryol::StaticArray<>).

In comparison the new C code has a much simpler data organization, there are
only 5 resource structs (buffer, image, shader, pipeline and pass), and
those are kept in 'handcrafted' resource pools (in C++ this would
typically be done with templates).

So it looks like most of the differences between the C++ and C code are around
the resource management code, this makes sense because the rendering code
which talks to OpenGL looks pretty much the same in both implementations.

The only big difference that comes to mind is the use of template container
code in the C++ code, which is most likely the main reason behind the
size difference. Not because C++ template code is inherently bigger then
similar C code, but because C++ containers where used in more places out of
convenience. Another size contributor is most like copy operations, in C this
is always guaranteed to be a simple memory copy, while C++ might need to
create complex code depending on the type being copied.

Another potential reason of why the C++ code could be bigger than similar
C code would be range checks in C++ container classes, but both in 
sokol_gfx as well as in the Oryol container classes all range check
asserts are removed in optimized builds, so this turned out a non-issue 
(whether removing range checks in release mode is a good idea is another topic).

I'll keep it at that for now. The plan was anyway to gradually replace the
platform wrapper parts in Oryol with Sokol C header (which can be more easily
integrated into existing projects than Oryol, which is more an integrated
framwork) and only keep a thin C++ 'convenience layer' over the C code. It
will be interesting to see how much more size reductions can be realized with
this strategy.

I think the main take-away is: while a carefully crafted C++ codebase should
*not* automatically result in bigger executables compared to implementing
the same functionality in C, it is easier in C++ to accidently introduce code
bloat by careless use of template code. And not because template code
is automatically bigger than 'handcrafted code', but because C++ template
classes are so convenient to use, instead of stopping and considering whether such a
'generic solution' is acceptable. Basically the typical 'if all you have
is a hammer, every problem looks like a nail' situation. Since C has no generics
(and no operator overloading), it is much harder to introduce accidential complexity
under the hood.

The situation is a bit like working around the garbage collector in
memory-managed languages. It is possible to write 'garbage free' code in GC
languages, but it requires detailed knowledge of how this specific garbage
collector works, and code to work around the garbage collector often looks
'non-idiomatic' (see asm.js for example). Achieving the same in a lower-level
non-garbage-collected language may actually result in less and cleaner code.
