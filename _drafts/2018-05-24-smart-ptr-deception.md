---
layout: post
title: "The Smart Pointer Deception"
---

_TL;DR_: why 'object handles' may be better suited for (reasonably) safe memory
management in C and C++ than raw or smart pointers

Modern C++ 'enthusiasts' often (rightfully) condemn the use of 'C style'
dynamic memory management with raw pointers and malloc (or "old-school C++"
new/delete) and recommend std library smart pointers like unique_ptr or
shared_ptr together with make_shared() or the new make_unique() as the magic
cure to all C++ memory management ailments.

After almost a decade in C++ smart-pointer-land I think now that smart
pointers are not much more than a hack: they immediately fix one type of
problem (potential memory corruption) and that's why they look pretty
nice at first. But smart-pointers become a dangerous trap in
non-trivial code bases because careless use of smart pointers will kill
performance, but not before they have infested the entire code base.

I think part of the problem is that languages with automatically managed
object references (Java, C#, and to some extent JS, ...) somehow made us
believe that it is possible to write reasonably-high-performance-code without
having to think about memory management.

However when you look behind the curtain the situation is a bit more difficult.
Both C# and JS are quite popular in game development (if you look outside 
the triple-A-box), and the web is full of advice of how to avoid 
garbage collection spikes and how to avoid garbage in the first place.
So while managed languages fix one class of problems (memory corruption),
it's harder to optimize for performance without a lot of mental effort.

It's not like the problem is limited to garbage collection though, ARC (Automatic
Reference Counting) in Objective-C also has a pretty terrible overhead, unless 
the programmer does an absurd amount of tweaking and hand-holding to a point
where explicitely freeing memory would most likely be the simpler option.

An interesting observation (especially for WebAssembly/asm.js) is that writing
'high performance code' in a high-level managed language often results in 
code that looks messier than doing the same in straight C, simply because
in the high-level language you need to leave the 'idiomatic' path and write
magic incantations to work around the language's built-in memory management
(however Unity's new C#/Burst dialect seems to explore an interesting
middle-ground, maybe performance-oriented coding *is* possible without
a footgun attached by choosing the right restrictions in a high-level language).

IMHO the *wrong* answer to automatic-memory-management problems is to create
faster-but-more-complex garbage collectors or general memory allocators (like
jemalloc). This just adds a lot of complexity in the wrong place.

I also think that the simplistic advice 'use C++ smart pointers instead of
raw pointers' ignores the actual problem: decentralized memory allocation and
freeing. A large code base where objects are created all over the place, and
destroyed whenever a smart pointer thinks it's the right time to do so is a
debugging- and performance-optimization hell, and for this problem smart
pointers add nothing over raw pointers. Shared pointers even make the problem
by obscuring where the deallocation actually happens. Good luck debugging a
memory leak with a giant spider web of objects pointing to each other through
smart pointers.

Such huge object spiderwebs, decentralized 'general' memory management, and
low performance go hand in hand. This sort of code does a lot of pointer
chasing, and causes lots of cache misses, resulting in a 'death by a thousand
cuts' performance profile, where the overall code runs slow without any
obvious performance hotspots to focus the optimization effort on. Smart
pointers are just the icing on the cake in this situation, they *do* help
with memory corruption issues, but they'll drag down performance even more,
and instead of memory corruption it's easy to introduce accidential memory
leaks (because some forgotten smart pointers might pin dead objects into
memory).

### How did we get in this mess?

A quick tour back into history to understand how we got there. Please note
that all this code is an example for what *NOT* to do! I'll get to the
better alternatives towards the end of the post.

## C with raw pointers and malloc/free

In traditional C, if you want to have a chunk of memory which outlives the
current function, you'd either make it a static global variable (which is 
terribly underrated since 'globals are bad' is the second-most cited cargo
cult after 'goto considered harmful', ignoring that it's only shared writable
globals that are bad. Global constants, or globals that are only visible in one
 translation unit are totally fine, even better than C++'s 
'private-but-still-exposed-to-the-world' concept.

The alternative to static globals is to allocate a piece of heap memory with
malloc() (or calloc()) and assign the result to a pointer, which - by convention - becomes
the owner of that piece of memory (where 'the owner' is responsible later to
free the memory, otherwise you'd get a memory leak).

After allocation, an initialization might be needed, so there's often an
initialization function (and vice versa for deinitialization):

```c
void fn() {
    item_t* item = (item_t*) malloc(sizeof(item_t));
    init_item(item);
    do_something_with_item(item);
    deinit_item(item);
    free(item);
}
```

One problem is that C tutorials often use such simplified examples to explain 
heap allocation, without a big fat "Danger!" sign and making it clear that such
'fine-grained' heap allocation is almost always a very bad idea. And maybe such
simple examples are the reason why heap allocation is used too carelessly.

But anyway, what are the typical problems with C-style malloc and raw pointers:

- forgetting to call the init() function results in use of garbage memory, leading
to hard-to-reproduce random bugs
- forgetting to de-initialize the 'object' before freeing the memory will
lead to memory leaks if init_item() does additional allocations
- use-after-free: after calling free, the pointer still contains a value but
no longer points to a valid item (this is also called a 'dangling pointer')
- double-free: if you handed the pointer to somewhere else, and there's confusion
about ownership (since it's just a convention, not enforced by the language)
two or more different places in the code might attempt to free the same memory chunk
- pointer arithmetic: if for some reason the code decides to
do fancy pointer arithmetic, the result may point into invalid memory regions
(which results in a 'buffer under- or overflow' if reading or writing through
the pointer)

Most of those problems can be caught with better tooling: static analyzers,
valgrind, or clang's address sanitizer. But of course it's better if those
problems are hard to introduce into the code in the first place.

## "old C++" with new/delete

Old school C++ improved the situation a bit by adding constructors and
destructors to the language:

```cpp
void fn() {
    item_t* item = new item_t("name", 23);
    do_something_with_item(item);
    delete item;
}
```
Now it's impossible to forget calling the initialization and deinitialization
functions because this is handled implicitely by the compiler, so it's less
likely to accidently work with unitialized memory, or introduce 'secondary
memory leaks' for stuff that's allocated during item initialization.

However this merging allocation and initialization has its downsides, there's
no control over how the memory is actually allocated and freed under the
hood. That's why placement-new has been added to C++ and explicitely calling
the destructor is possible:

```cpp
void fn() {
    void* ptr = malloc(sizeof(item_t));
    item_t* item = new(ptr) item_t("name", 23);
    printf("%p\n", item);
    item->~item_t();
    free(item);
    return 0;
}
```

But for many memory corruption issues, this new/delete with raw
pointers doesn't help much, all of the following problems are
still possible:

- use-after-free
- double-free
- buffer under- and overflow
- 'primary memory leaks' (forgetting to call delete)

## Modern C++ (smart pointers)

...so the new old thing in C++ is to use the std library smart pointers, 
especially with the introduction of unique_ptr in C++11 and make_unique in C++14:

```cpp
void fn() {
    auto item = std::make_unique<item_t>("name", 23);
    do_something_with_item(item);
}
```

This fixes nearly all the remaining problems:

- (undetected) use-after-free
- double-free
- pointer arithmetic not allowed on smart pointers
- 'obvious' memory leaking not possible, but it's possible that a 'forgotten
smart pointer' pins its object into memory, which technically also a memory leak.

## Rejoice?

Yay! Modern C++ fixes all the pesky memory corruption problems for us,
we no longer need to care about memory management, just like them fancy 
garbage collected languages \o/

So let's rejoice and rewrite our million-line object-spiderweb code base to
C++14 and everybody will be happy!

Well not really, the refactoring energy would be put to better use replacing
the spiderweb architecture with a proper data-oriented architecture first. As
a side effect, pointers will become much less important, and smart pointers
will entirely become...uh, 'pointless'.

## Handles are the better Pointers

Let me explain: the first idea is to get rid of the 'decentralized memory
management' and move it completely inside centralized systems (e.g.
rendering, physics, AI, whatever...). Those systems know better how to
manage their memory instead of going directly through a general allocator in
thousands of places all over the code base.

The system remains the sole owner of the memory it manages, since it also
knows best how to free (or reuse) the memory (for instance in modern 3D-APIs,
rendering resources cannot be destroyed immediately after the user code
no longer needs them, the rendering system must delay the destruction up
to a few frames until it is guaranteed that the GPU no longer accesses the
resource).

Construction and especially destruction is explicit, for instance creating
and destroying a texture in [sokol_gfx.h](https://github.com/floooh/sokol)
looks like this:

```c
sg_image img = sg_make_image(&img_desc);
...
sg_destroy_image(img);
```

Note that _img_ is not a pointer, but an opaque handle.

Now the first breakthrough: with systems having complete control over their
memory management, the best and easiert solution is often to arrange
'objects' into simple arrays. Those can be pre-allocated and fixed-size, or
grow dynamically, the outside world isn't concerned about such details. With
all the active objects packed tightly into arrays we have a good foundation
for avoiding cache misses (it's not quite so easy, more on that later).

In the most simple case, a fixed maximum number of living objects is acceptable,
in this case memory allocation only needs to happen during initialization,
and the system can be completely allocation-free for the entire time the application
is running. Suddenly a completely game loop which doesn't allocate or free 
doesn't sound so crazy!

But even when the maximum number of objects is unbound, having all
allocations for a system in one place is much better for debugging memory
related problems, and memory allocation behaviour can be tweaked and
specialized exactly for the needs of a specific system.

With these changes in place we have better control over memory-related 
performance problems:

- memory allocator performance and address space fragmentation is no longer
an issue, in the best case we only allocate once during system
initialization, and free once during system shutdown, and even with growing
arrays dynamically, memory allocation will be very infrequent. Ooops, suddenly
jemalloc, dlmalloc and all the other super-advanced general allocators are obsolete...
- packing live objects tightly into arrays is a good start to improve cache-hit-rate
(as long as the objects are not too big, but nothing is stopping a system
from splitting such an 'object' into several smaller data items which contain
just the data needed for a specific processing step)

But now we might run into problems if we hand pointers to the outside world:

- if our system-internal arrays are growing, they'll change their location in
memory, all the pointers that had been handed out so far will point to the
wrong location now...
- if the system splits complex objects into smaller 'data items', there isn't
really a single address location which uniquely identifies an object, unless
we have a 'master object' which keeps track of all its data items... hrmpf.
- with 'pointers as public object refs' memory corruption problems like use-after-free,
or double-free might still be a problem, even worse so when the same item-slot
has already been reused for another object

What about smart pointers? They were at least a good solution for memory corruption
issues, so why not use them now? The problem with the default behaviour of smart
pointers is that they expect to own the memory they point to. This clashes with
the idea that centralized systems do their own memory management. We could start
to doctor around the symptoms, like attaching a 'custom deleter' to the smart
pointers, but all this is in vain when the system decides to grow an internal array,
because all smart pointers out in the wild would point to the wrong memory locations.

This is basically the reason why smart pointers are 'pointless' with a
data-oriented approach. And this approach has so far fixed a few important 
problems (much fewer allocations, nor more address-space fragmentation and
potentially better performance due to fewer cache misses).

So what if we dropped pointers alltogether and moved to handles?

The most simple and obvious type of handle is a simple array index.

What problems would such a simple array index handle solve?

- Memory corruption problems by 'user code' would essentially be eliminated,
there are no pointers, thus no way to access memory, thus no accidential
memory corruption ;)
- Growing the system-internal arrays or generally moving
the array around in memory is no longer a problem, since the base address of
the array remains private to the system
- Use-after-free can partially be deteced, if a 'freed' array slot
contains a flag whether it is currently valid.

There's a problem if the same slot has been reused for a new object though,
consider this code:

```c
sg_image A = sg_make_image(...);
// A is slot 1
sg_destroy_image(A);
// slot 1 is free now, and reused for B
sg_image B = sg_make_image(...)
// B is slot 1, but we're trying to destroy A, which is still also '1'
sg_destroy_image(A);
```

If the sg_image handle ist just an index, the double-free in the last line
cannot be detected as bug, since A still points to slot 1 (which is now
occupied by image B), image B is accidently destroyed and the system
has no way to detect the double-free bug...

## Catching Dangling Handles

The solution to detect dangling handles (an old handle pointing to a 
freed or reused item slot) is to use a few bits in the handle for a
'unique stamp'. Let's say we have 32-bit handles, but we only need
up to 4096 objects that are live at the same time. We only need 12 bits
to index into an array with 4096 entries, which leaves the remaining
20 bits free for a unique-stamp:

```
+--------------------+------------+
|    unique stamp    | array index|
+--------------------+------------+
```
We also need a special 'invalid handle' value, which must be reserved and
not returned to the outside world when creating a new object.

Every time a new object is created, a 'pseudo-unique' handle is generated
from a 'free slot index' which will be put into the handle's lower bits, and
the current value of a 'unique counter' that's incremented for each new
handle (and wraps around when it overflows). Ideally, each new handle is
unique, even for array slots that have been freed and reused. However the
simple approach described here can lead to duplicate handles after the unique
stamp overflows, if by accident a unique-stamp/array-index pair is generated
which was already used before. I haven't come up with a waterproof solution
to this problem yet, except assigning as many bits to the unique-stamp part
as possible. There are probably a few good tricks to delay such a handle
clash as much as possible, but so far the problem wasn't important enough, so
\*shrugs\*

Once a system hands out such unique handles, it's easy to detect dangling access:

- whenever a new object is 'created' (by picking a free array slot), the new
handle is stored in this slot, and returned to the outside world
- whenever an object is destroyed, the 'invalid handle' is stored in
the array slot that just became vacant
- whenever an 'untrusted' handle is coming in from the outside world, it
is compared to the handle stored at the associated slot index, if both
are identical, the handle and array item still match and the operation
is valid, otherwise the array item belonging to the handle had been
freed or reused in the meantime.

[TODO: handing out temporary pointers]


[TODO: dynamic type information vs static typing]

