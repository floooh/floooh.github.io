---
layout: post
title: "WebAssembly Demystified"
---
My feeble attempt at a non-too-technical, mostly FUD- and hyperbole-free
FAQ about WebAssembly.

#### Is WebAssembly CPU machine code?

No, WebAssembly is an intermediate format more akin to
.NET or Java bytecode than machine code. If you don't like
the .NET or Java association, just think of it as an instruction
set of a 'fantasy CPU'.

#### Is WebAssembly assembly code?

No, 'assembly code' usually means the text representation of a specific
CPU instruction set. WebAssembly in its typical form is neither a text
representation nor is it CPU-specific (the WASM spec *does* define a text
representation though, which is basically WebAssembly-assembly).

#### Is WebAssembly "Java Applets/ActiveX/Plugins 2.0"?

No:

- it is different from **Java Applets** because it reuses the
existing Javascript VM that already exists in browsers instead
of adding another big VM which would have to be security-hardened. WebAssembly
also doesn't allow to call out into native code like Java Applets did via JNI.

- **ActiveX and browser plugins** were native machine code DLLs where the security was
built on trusting a 3rd-party and wishful thinking. With WebAssembly
the trust rests solely on the browser maker and his ability to secure the
browser sandbox.

#### Is WebAssembly a security risk?

No, unless you consider Javascript a security risk. WebAssembly is about as
secure as Javascript since it runs in the same sandbox. Typical WebAssembly
implementations only add a fairly small parser module to the existing JS
engine. For instance in [WebKit](https://github.com/WebKit/webkit), WebAssembly seems to
account for about 10k out of 432k lines of code for the whole JavaScriptCore
engine. So only about 2% of the whole Javascript engine are a new 
WebAssembly-specific attack surface.
 
#### Can I access sockets, files, native API X... from WebAssembly?

No, WebAssembly runs inside the browser sandbox and can only access Web
APIs (just like Javascript). However, the emscripten SDK provides wrappers
for many common native APIs which simplifies porting existing C/C++ code
over (for instance sockets-to-WebSockets, GL-to-WebGL, OpenAL-to-WebAudio,
and some more...).

#### Can I load and call into a DLL from WebAssembly?

No, DLLs are native machine code and thus off-limits. 

Slightly related: WebAssembly supports _dynamic linking_ between WASM modules,
this allows to reuse shared code modules (the idea is to reduce download size
by moving shared code into a separate module which can be cached locally).

#### Can I start a program or process from WebAssembly?

Nope, same reason as DLLs, it would be unsafe.

#### Can WebAssembly be used to circumvent adblockers?

No, it can only do things that Javascript can do. Even if the ad doesn't
download images but renders its content 'procedurally' through
WebAssembly code, the actual WebAssembly module must have been downloaded
from some URL first (usually an ad-network server) and this can be
intercepted and blocked by adblockers just like regular ads.

#### ...but what about 'Show Source'?!

Come on now... that train had already left since people started using
Javascript minifiers and obfuscators. Just print a link to the github repo on
the JS console.

#### As a frontend developer, do I need to learn C/C++ now?

No. One interesting use-case for WebAssembly is to create libraries that are
callable from Javascript. Computation-heavy code (for instance image
manipulation, physics engines, or whole game engines) could be implemented
as WebAssembly modules, but offer a Javascript API. _Using_ such WebAssembly
module wouldn't be much different from using a Javascript framework.

#### Can WebAssembly manipulate the DOM?

For some reason this question seems to pop up most frequently :) The
answer is both no and yes. It cannot *directly* access the DOM, but it can
call out into Javascript, and JS can then work on the DOM. In emscripten this
is as simple as directly inlining Javascript code into C++ code. But jumping
back and forth between WebAssembly and Javascript isn't fast (but it isn't
terribly slow either, as demonstrated by emscripten's GL-to-WebGL wrapper).
It's perfectly possible to create a "dom.h" C or C++ library to do typical
"frontend programming" via WebAssembly, but this doesn't make sense if the
motivation is performance, and C/C++ is probably not the nicest language for
this. DOM manipulation might make more sense with other languages that are
meant as a Javascript replacement.

#### Isn't WebAssembly useless for garbage collected languages?

YES AND THANK THE GODS FOR THAT! Erm, actually no, scratch that :)

It is true that WebAssembly doesn't have garbage collection support, but this
doesn't mean that a WebAssembly module cannot implement its own garbage
collector. Look at [Unity3D's
IL2CPP](https://blogs.unity3d.com/2015/07/09/il2cpp-internals-garbage-collector-integration/), [Kotlin
Native](https://blog.jetbrains.com/kotlin/2017/04/kotlinnative-tech-preview-kotlin-without-a-vm/),
or clang's [Automatic Reference
Counting](https://clang.llvm.org/docs/AutomaticReferenceCounting.html) for
examples of garbage-collected languages that don't need a garbage-collected VM to work.
There is some info about potential GC support in a future WebAssembly version [here](https://github.com/WebAssembly/design/blob/master/GC.md).

#### But WebAssembly programs are too big!

This is not the fault of WebAssembly itself, which is a very compact byte code
format, but usually is a problem of the code base that's been compiled to WebAssembly.

There are several reasons why WebAssembly modules can become bloated:

- **Language runtimes**: Most languages need some sort of runtime library to be
useful. In C this includes functions like printf, strcmp, fopen, ... in C++
the std library classes can be considered the runtime even though most std
library code is in inline methods. Higher level languages like C# have a much
'richer' runtime, this is good for programmer productivity, but terrible for
code size.
- **Huge 'Modulith' Projects**: Large C++ projects like AAA game engines or big
UI frameworks are easily multi-million-line code bases, and also often
heavily use OOP features like dynamic dispatch (virtual methods). Such code
bases make it very hard for the compiler's dead code elimination to do its
thing.
- **3rd-party-dependencies**: Many libraries and (for instance) game
development middleware haven't been designed with binary size in mind, and
adding such dependencies carelessly can quickly grow out of control.

In the 'native world' executable sizes of several dozen megabytes are not
uncommon these days, but on the web this is simply not acceptable. The TL;DR
is to use a more 'embedded-programming' philosophy when writing code for
WebAssembly (also here's one of my older blog posts about the topic: 
[10 simple diet tricks for asm.js](http://floooh.github.io/2016/08/27/asmjs-diet.html)).

WebAssembly programs *can* be small, but it takes effort. Hopefully now that
'executable size' is important again, we will also see more developers focus on this. Maybe
somebody comes up with a smallest-possible C runtime (I'm sure there's still
room for improvements if seldom-used features are removed).


#### If WebAssembly is running in the JS engine, how can it be fast?

...mainly for the same reasons that asm.js is faster than 'idiomatic' Javascript:

- **No garbage collection**: both asm.js and WASM do not produce 'garbage' 
that needs to be collected, thus the JS garbage collector doesn't kick in while
asm.js or WASM is running.
The only exception is when calling out into Javascript to talk to web APIs,
the JS shim usually needs to create temporary JS objects, and this produces 
garbage. New web APIs often make it a point to
offer 'garbage free' functions, which will help a lot when being called
from asm.js or WebAssembly.

- **Everything is static**: Javascript is an extremely dynamic language, which requires a
lot of effort to make it run fast since the JS engine needs to track type
changes and may decide to re-compile and re-optimize functions during runtime.
For highly dynamic code this burns a lot of CPU cycles and makes performance
behaviour unpredictable. WASM and asm.js both got rid of all this 'dynamicism'. 
Once the code is compiled, it is guaranteed to stay that way.

- **A linear memory model**: This is probably the biggest difference to most
'managed VMs', and the most important factor for performance. High level
languages like Javascript, C# or Java treat memory as an idealized,
infinitely fast and infinitely big resource which the programmer shouldn't
have to think about. Unfortunately the real world is a bit more complicated,
for instance there are little annoying details like cache-misses. In short,
having full control over the exact layout of data in memory is essential for
good performance. System programming languages like C, C++ and Rust allow
this fine level of control, and one of the most ingenious features of
emscripten (even before asm.js) was that it moved this 'linear memory model'
into the Javascript environment by putting the C heap into a single Javascript 
TypedArray. This way the memory layout defined by the original C/C++ source code
is preserved, and the advantages of arranging
data in a cache-friendly way carry over.

That's about all the FAQ-worthy questions I can think of for now. Until next time :)





