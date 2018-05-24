---
layout: post
title: "The Smart Pointer Deception"
---

_TL;DR_: why 'object handles' are better suited for (reasonably) safe memory
management in C and C++ than raw or smart pointers (if you've been following
the data-oriented-philosophy or ever heard of Mike Acton all of this will be
fairly boring and obvious I'm afraid)

Modern C++ 'enthusiasts' often (rightfully) condemn the use of 'C style' dynamic
memory management with raw pointers and malloc (or "grandpa's C++" new/delete)
and recommend std library smart pointers like unique_ptr or shared_ptr
(together with make_shared() or the new make_unique() to create objects) as the
magic cure to all C++ memory management ailments.

I want to argue that smart pointers are not much more than a hack
which immediately fix one type of problem (potential memory corruption),
but are a dangerous long-term trap in non-trivial code bases because
careless use of smart pointers will kill performance, but not before they
have infested the entire code base.

I think the core of the problem is that languages with automatically 
managed object references (Java, C#, JS, ...) somehow made us believe
that it is possible to write reasonably-high-performance-code without
having to think about memory management.

However when you look behind the curtain the situation is difficult.
Both C# and JS are quite popular in game development (if you look outside 
the triple-A-box), and both languages have to fight with peformance problems
caused by the 'built-in' memory management for heap objects, and the information
on the net how to 'work around the garbage collector' is legion (it's
interesting that the techniques to write 'GC friendly code' for managed
languages are the same as for C/C++: reuse objects through pooling,
or even better, don't use heap objects at all, and keep 'hot data' in
simple arrays).

It's not like the problem is limited to garbage collection though, ARC (Automatic
Reference Counting) in Objective-C also has a pretty terrible overhead, unless 
the programmer does an absurd amount of tweaking and hand holding to a point
where explicitely freeing memory would result in simpler code.

An interesting observation (especially for WebAssembly/asm.js) is that writing
'high performance code' in a high-level managed language often results in 
code that looks more messy than doing the same in straight C, simply because
in the high-level language you need to leave the 'idiomatic' path and write
magic incantations to work around the language's built-in memory management.

IMHO the *wrong* answer to those problems is to create faster-but-more-complex
garbage collectors or general memory allocators. This just adds a lot of
complexity in the wrong place.

I also think that the simplistic advice 'use smart pointers instead of raw
pointers' ignores the actual problem: decentralized memory allocation and
freeing. A large code base where objects are created all over the place, and
destroyed whenever a smart pointer thinks it's the right time to do so is a
veritable debugging and performance optimization hell, and for this problem
smart pointers add nothing over raw pointers. They even make the problem
worse by hiding the deallocation. Good luck debugging a memory leak with
a giant spider web of objects pointing to each other through smart pointers.

Such huge object spiderwebs, decentralized 'general' memory management, and low
performance go hand in hand. This sort of code does a lot of 'pointer chasing',
causing lots of cache misses, and resulting in a 'death by a thousand cuts' 
performance profile, where the overall code runs slow without any
obvious performance hotspots to focus on. Smart pointers are just the icing on the cake
in this situation, they *do* help with memory corruption issues, but they'll 
drag down performance even more, and instead of memory corruption it's easy to
introduce accidential memory leaks (because some forgotten smart pointers
might pin dead objects into memory).

### How did we get in this mess?

A quick tour back into history to understand how we got here. Please note
that all this code is an example for what *NOT* to do! I'll get to the
better alternatives towards the end of the post.

## The Old Kingdom (raw pointers and malloc/free)

In traditional C, if you want to have a chunk of memory which outlives the
current function, you'd either make it a static global variable (which is 
terribly underrated since 'globals are bad', which is right after 'goto considered
harmful' as far as programmer cargo cults go).

The alternative (which is often recommended too lightly) is to allocate a piece
of heap memory with malloc() and assign the result to pointer, which - by convention -
becomes the owner of that piece of memory (where 'the owner' is responsible later
to free the memory, otherwise you'd get a memory leak).

Since malloc() returns uninitialized memory, the alternative is to call calloc()
to get zero-initialized memory and/or call an initialization function:

```c
void fn() {
    item_t* item = (item_t*) malloc(sizeof(item_t));
    init_item(item);
    do_something_with_item(item);
    deinit_item(item);
    free(item);
}
```

Of course in this case the infinitely better option is to not heap-allocate at all:
```c
void fn() {
    item_t item;
    init_item(&item);
    do_something_with_item(&item);
    deinit_item(&item);
}
```
...in fact it's quite hard to justify heap allocation for simple example code,
maybe that's one of the reason why heap allocation is used so carelessly,
since simple examples may give the impression that allocation is cheap and simple
too...

But anyway, what are the typical problems with C-style malloc and raw pointers:

- using uninitialized memory (if forgetting to call init_item() after malloc())
- forgetting to de-initialize the 'object' before freeing the memory, this will
lead to memory leaks if init_item() does additional allocations
- use-after-free: after calling free, the pointer still contains a value, that
no longer points to a valid item though (this is also called a 'dangling pointer')
- double-free: if you handed the pointer to somewhere else, and there's confusion
about ownership (since it's just a convention, not enforced by the language)
two or more different places in the code might attempt to free the same memory chunk
- pointer arithmetic: if for some reason the code decides to
do fancy pointer arithmetic, the result may point into invalid memory regions
(that's usually called a buffer under- or overflow)

Most of those problems can be caught with better tooling: static analyzers,
valgrind, or clang's address sanitizer. But of course it's better if those
problems are hard to introduce into the code in the first place.

## The Middle Kingdom (C++ and new/delete)

Old school C++ improved the situation a bit by adding constructors and
destructors to the language:

```cpp
void fn() {
    item_t* item = new item_t;
    do_something_with_item(item);
    delete item;
}
```
Now it's impossible to forget to call the initialization and deinitialization
functions because this is handled implicitely by the compiler, so it's less
likely to accidently work with unitialized memory, or introduce 'secondary
memory leaks' for stuff that's allocated during item initialization.

However this is a double-edged sword. Merged allocation/initialization and
deinitialization/deallocation means that we don't have much control over
how the memory is actually allocated/freed under the hood. That's why 
placement-new exists and explicitely calling the destructor is 
possible:

```cpp
void fn() {
    void* ptr = malloc(sizeof(item_t));
    item_t* item = new(ptr) item_t;
    printf("%p\n", item);
    item->~item_t();
    free(item);
    return 0;
}
```

But for many memory corruption issues, this doesn't help:

- use-after-free
- double-free
- buffer under- and overflow
- 'primary memory leaks' (forgetting to call delete)

...all possible and just as likely as before

## The New Kingdom (smart pointers and RAII)

...so the new old thing in C++ is to use the std library smart pointers, 
especially with the introduction of unique_ptr in C++11 and make_unique in C++14:

```cpp
void fn() {
    auto item = std::make_unique<item_t>();
    do_something_with_item(item);
}
```

This fixes nearly all the remaining problems:

- (undetected) use-after-free no longer possible
- double-free no longer possible
- pointer arithmetic only possible with a lot of criminal energy
- 'obvious' memory leaking not possible, but it's possible that a 'forgotten
smart pointer' pins its object into memory, which is basically also a memory leak

## Rejoice?

Yay! Modern C++ fixes all the pesky memory corruption problems for us,
we no longer need to care about memory management, just like them fancy 
garbage collected languages!

Let's lift our brittle millions lines object spiderweb code base to C++14 and
all our problems will go away!

Well no, the refactoring energy would be put to better use replacing the
spiderweb architecture with a proper data-oriented approach, where 
memory management is taken care of in central systems. As a side effect,
pointers will become much less important, and smart pointers will become 
entirely ... 'pointless'.

## Handles are the better Pointers

Let me explain: the idea is first to get rid of the 'decentralized memory
management' and move it completely inside centralized systems (e.g.
rendering, physics, AI, whatever...), since those systems know better how to
manage their memory instead of going directly through a general allocator in
thousands of places all over the code base.

The system remains the sole owner of the memory it manages, since it also
knows best when it is a good time to free memory (or simply reuse it). 

Now the first breakthrough: with systems having complete control over their
memory management, the best and easiert solution is often to arrange
'objects' into simple arrays. Those can be fixed-size or grow as needed,
the outside world shouldn't be concerned about this. With live objects
packed tightly into arrays, we have a good foundation for avoiding cache
misses (it's not quite so easy, more on that later).

In the most simple case, a fixed maximum number of living objects is acceptable,
in this case memory allocation only needs to happen during initialization,
and the system can be completely allocation-free for the entire time the application
is running. Suddenly a completely allocation-free game loop doesn't sound
so crazy anymore!

But even when the maximum number of objects is unbound, having all allocations
for a system in one place is much better for debugging memory related problems,
and memory allocation behaviour can be tweaked and specialized exactly for the needs of
a specific system.

With these changes in place we have better control over memory-related 
performance problems:

- memory allocator performance and address space fragmentation is no longer
an issue, in the best case we only allocate once during system
initialization, and free once during system shutdown, and even with growing
arrays memory allocation will be very infrequent. Ooops, suddenly
jemalloc, dlmalloc and all the other super-advanced general allocators obsolete...
- packing live objects tightly into arrays is a good start to fix cache misses
(as long as the objects are not to big, but nothing is stopping a system
from splitting the 'object' into several smaller data items now which contain
just the data needed for a specific processing step)

But now we might run into problems if we hand pointers to those system
objects out into the wild:

- if our system-internal arrays are growing, they'll change their location in
memory, all the pointers that have been handed out so far will point to the
wrong location now...
- if the system splits complex objects into smaller 'data items', there isn't
really a single address location which uniquely identifies an object, unless
we have a 'master object' which keeps track of all its data items... too complicated!
- with 'pointers as public object refs' memory corruption problems like use-after-free,
or double-free might still be a problem, even worse so when the same item-slot
has already been reused for another object, which is very likely for such 
an object-pool system... debugging this would be a nightmare...

What about smart pointers? They were at least a good solution for memory corruption
issues, so why not use them now? The problem with the default behaviour of smart
pointers is that they expect to own the memory they point to. This messes with
the idea that centralized systems manage and own their own memory. We could start
to doctor around the symptoms, like attaching a 'custom deleter' to the smart
pointers, but all this is in vain when the system decides to grow an internal array.

This is basically the reason why smart pointers are 'pointless' with a
data-oriented approach. And this approach has so far fixed a few important 
problems (much fewer allocations and potentially better use of CPU caches). 

So what if we dropped pointers alltogether and moved to handles?

The most simple and obvious type of handle is a simple array index into a
system-internal item array (or a single index into multiple arrays if an
'object' is split into several smaller data items).

What problems would such a simple array index handle solve?

- memory corruption problems by 'user code' would essentially be eliminated,
there are no pointers, thus no way to access memory, thus no accidential
memory corruption ;)
- growing the system-internal arrays or generally moving
the array around in memory is no longer a problem, since the base address of
the array remains private to the system

[TODO: remaining problems: double-free, dangling]

[TODO: handing out temporary pointers]

[TODO: dangling protection with a 'unique count']

[TODO: dynamic type information vs static typing]

