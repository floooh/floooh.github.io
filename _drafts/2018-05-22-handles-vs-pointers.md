---
layout: post
titles: Handles as smart pointer alternatives
---

A quick rundown of how I'm using handles as an alternative to raw- or smart-
pointers. The whole thing is the result of a slow change of how I think about
memory management (from "everything is an object on the heap" to "(nearly) everything
is an array item").

I think this approach has (at least) the following advantages over raw or smart-pointers:

- provides roughly the same protection against memory corruption issues as C++ smart pointers (of course running the code through static analyzers and clang address sanitizer is still a good idea)
- explicit control over memory allocation without the hassle of C++ custom allocators
- memory leaks are caught before the process runs out of memory and are easy to debug (in the sense of "what caused the leak")
- detection of 'dangling pointer protection' (or rather 'dangling handles')
- simple ownership concept (no shared vs unique, no ownership transfer, etc...)
- works equally well for C and C++
- unlike pointers, handles can easily be serialized or shared with other processes

Handles only really make sense with a radical change of how heap objects are
allocated though: instead of 'decentralized, general-purpose allocation
(make_shared or malloc all over the place)', all heap allocations (and freeing)
should happen in centralized systems (not in *one* centralized system, but
several, for instance a rendering system knows best how textures and
buffers are created, and this may be different from the ideal case how
rigid-bodies are created in a physics system).

Incidentally, this explicit and centralized memory management approach is a
very good fit for a data-oriented design philosophy, where 'data items' are
usually grouped in simple liner arrays, instead of being spread all over the
address space (...and std smart pointers with make_shared are much less
useful in this situation, because they want to control the heap allocation,
...unless custom allocators yadda yadda...).
