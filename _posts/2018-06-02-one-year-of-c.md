---
layout: post
title: "One year of C"
---

It's now nearly a year that I started writing non-trivial amounts of C code
again (the first sokol_gfx.h commit was on the 14-Jul-2017), so I guess it's
time for a little retrospective.

In the beginning it was more of an experiment: I wanted to see how much
I would miss some of the more useful C++ features (for instance namespaces, function
overloading, 'simple' template code for containers, ...), and whether it is
possible to write non-trivial codebases in C without going mad.

Here are all the github projects I wrote in C:

- **[sokol](https://github.com/floooh/sokol)**: a slowly growing set of 
platform-abstraction headers
- **[sokol-samples](https://github.com/floooh/sokol-samples)** - examples
for Sokol
- **[chips](https://github.com/floooh/chips)** - 8-bit chip emulators
- **[chips-test](https://github.com/floooh/chips-test)** - tests and examples for the chip-
emulators, including some complete home computer emulators (minus sound)

All in all these are around 32k lines of code (not including 3rd party code
like flextGL and HandmadeMath). I think I wrote more C code in the recent 10
months than any other language.

So one thing seems to be clear: yes, it's possible to write a non-trivial
amount of C code that does something useful without going mad (and it's even
quite enjoyable I might add).

Here's a few things I learned:

### Pick the right language for a problem

This is part of a decade-long personal transformation: instead of trying to
solve every problem with a single language (in my case: C++), and become an
entrenched 'C++ expert', it is much more enjoyable and productive to learn a
few different languages and pick a language that naturally fits a problem.
Putting C back into my language toolbox was a good decision for problems
where C++ was overkill (although I didn't realize that C++ is overkill for
many problems before I started writing C again). C fits into a multilanguage
toolbox better than C++ because integrating C with other languages is usually
much simpler than trying the same with C++.

Here's what my current language toolbox looks like:

- **python**: for cross-platform shell-scripting stuff, command-line tools
where performance doesn't matter, or generally glueing together several
tools and applications (e.g. tools like Maya or Blender are python-scriptable, 
I wish more UI application were)
- **Typescript**: for anything web-front-end related and where more than a few
lines of Javascript is needed
- **C**: my first choice now for writing libraries and any sort of
'building blocks' code
- **C++**: simple 'Orthodox C++' is still useful for bigger code bases, and
of course when depending on other code that's written in C++ (like Dear ImGui
or SoLoud). I have no intention to go 'all Modern C++' though. Picking the 
right language subset is even more important than in the past.

These are my bread-and-butter languages where I have written the most code in,
unfortunately I didn't have much need for **Go** yet, I would use this for
anything running on a server backend. I'm also keeping an eye on a number of 
small 'better C' languages that are starting to appear, like 
[Nim](https://nim-lang.org/), [Zig](https://github.com/ziglang/zig) or 
[Ion](https://github.com/pervognsen/bitwise). Personally I think these
small language are much more exciting than big oil tankers like
Rust or Swift.

I guess the paradox here is that it's better to have a shallow knowledge of
multiple simple languages than a deep knowledge of a single complex language
;)

### C is a perfect match for WebAssembly

What caught me a bit by surprise is how much smaller WebAssembly demos
written in C are when compared to similar demos written in C++.

For instance the sokol-gfx triangle sample is a 22 KByte download (9 KB
WASM, and 12.9 KB JS, all compressed). Before that I was quite convinced that
an emscripten WebAssembly demo using WebGL can't go below around 90 KByte (compressed)
(also see this [blog post](http://floooh.github.io/2018/05/01/cpp-to-c-size-reduction.html)).

### C99 is a huge improvement over C89

At one point I thought that it would be a good idea to make the sokol-headers
fully C89 standard compliant, but I soon discovered that what I knew as C
since the middle of the 90's wasn't actually proper C89. Even before C99, all
compilers started to add extensions that made C89 more friendly (like
declaring variables anywhere, winged comments, or "for (int...)" loops), in
the end I decided that it really wasn't worth it to make the code C89
compliant until there's a real-world use case where C89 is really required.

Instead the headers now use a subset of C99 that compiles both in C and 
C++ mode on the 3 major compilers (gcc, clang and cl.exe).

The biggest improvement that C99 brings to the table is easily **designated
initialization**, I think I never saw such a simple and elegant extension
to an existing language that is so useful (it puts **all** 
the different ways to initialize a struct or object C++ came up with 
over time to shame).

### The dangers of pointers and explicit memory management are overrated

This statement comes with a big caveat: Careful API design.

Pointer- and allocation-free programming is an interesting topic
for its own blog post (but also hard to put into a single post as
the huge pile of discarded drafts shows).

To make a long story short: yes, raw pointers as _owners of heap objects_
are dangerous, and C++ smart pointers can help with this problem. But 
pointers as owner of an allocation are a broken concept to begin with,
and smart pointers are only a half-assed workaround for the underlying
problem (which is decentralized ownership).

Have a look at this sokol-gfx example:

[https://github.com/floooh/sokol-samples/blob/master/html5/triangle-emsc.c](https://github.com/floooh/sokol-samples/blob/master/html5/triangle-emsc.c)

The sokol-gfx API doesn't return any pointers, and pointers going into the
API are always 'borrow references', sokol-gfx will only inspect the data and
copy what needs to persist, but never take ownership of the data. There are
no calls to malloc or free anywhere (if the code were C++ there wouldn't be
any smart pointers, new/delete or make_shared/make_unique either).

Even internally there's very little to no dynamic memory management going on
while the application runs (depending on the 3D API backend).

In the 32 kloc of C code I've written since last August, there are only
13 calls to malloc overall, all in the sokol_gfx.h header,
and 10 of those calls happen in the sokol-gfx initialization function.

The entire 8-bit emulator code (chip headers, tests and examples, about
12 kloc) doesn't have a single call to malloc or free.

So with a bit of care when building APIs, C code doesn't have to be
riddled with pointers or malloc/free calls. 

### Less Boilerplate Code

This is a bit weird, but when writing C code I spent less time writing
pointless boilerplate compared to my typical C++ code. Writing C++ classes
often involves writing constructors, destructors, assignment- and
move-operators, sometimes setter- and getter-methods... and so on. This is so
normal in C++ that I only really recognized this as a problem when I noticed
that I didn't do this in C.

C doesn't have RAII, which at first seems like a disadvantage to C++. But
it's only really a problem when trying to write C code like C++. Instead if C
is used like the gods intended (all data is POD, copying can be done with a
simple memory copy, and no actions need to happen on destruction), all the
code for construction, destruction and copy/move operators isn't needed in
the first place! This sort-of requires dropping the idea of 'pointers as
owners' as well, but as shown above that's a bad idea anyway.

The only 'boilerplate' I have in my C code is where I need to replace
zero-initialized struct members with default values. Maybe
allowing to define default values in struct declarations would be a useful
addition to the language.

### Less Language Feature 'Anxiety'

This is also a bit strange, but I feel more calm and focused when writing C
code. Again this is something that I would never have noticed without
starting to write C again. When writing C++ there's always more than one way
to do something and many micro-decisions need to happen:

Do I wrap this concept in a class or are namespaced functions better? Does
the class need constructors? Does it need a copy constructor? Multiple copy
constructors? A move constructor? What constructors need to be explicit? What
type of initializations make sense? Initializer lists? Constructors with
default parameters? Multiple overloaded constructors? Geez, and that's
just for the initialization topic...

As a C++ programmer I developed my own pet-coding-patterns and bad behaviours
(e.g. make methods or destructors virtual even if not needed, create objects
on the heap and manage them through smart pointers even if not needed, add a
full set of constructors or copy-operators, even when objects weren't 
copied anywhere, and so on). All of this is obviously bad, but it's some
sort of automatic coping mechanism to deal with the complexity of C++.

Since C is such a simple language, most of those micro-decisions simply don't
exist once your mind has tuned itself to do things the C way instead of trying
to write C++ code in C.

When writing C code I have the impression that each line of code does something
useful, and I worry less about having selected the right language feature.

### Conclusion

All in all my "C experiment" is a success. For a lot of problems, picking
C over C++ may be the better choice since C is a much simpler language (btw,
did you notice how there are hardly any books, conferences or discussions
about C despite being a fairly popular language? Apart from the neverending
bickering about undefined behaviour from the compiler people of course ;) 
There simply isn't much to discuss about a language that can be
learned in an afternoon.

I don't like some of the old POSIX or Linux APIs as much as the next guy
(e.g. [ioctl()](http://man7.org/linux/man-pages/man2/ioctl.2.html), the
[socket API](http://man7.org/linux/man-pages/man2/select.2.html) or some of
the [CRT library
functions](http://www.cplusplus.com/reference/cstring/strtok/)), but that's
an API design problem, not a language problem. It's possible to build
friendly C APIs with a bit of care and thinking, especially when C99's
designated initialization can be used (C++ should *really* make sure that the
*full* C99 language can be used from inside C++ instead of continuing to
wander off into an entirely different direction).
