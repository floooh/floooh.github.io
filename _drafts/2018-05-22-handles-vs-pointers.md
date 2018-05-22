---
layout: post
titles: Better memory management with handles
---

TL;DR: why object handles may be the better option to both raw- and smart-pointers
for typical memory-management- and -corruption problems in C and C++, especially
together with a data oriented design. Probably not very interesting for
veterans who have run into those problems before, but hopefully useful for
beginner- and intermediate-programmers.

### Smart Pointers vs. Data-Oriented-Design

'Modern C++ enthusiasts' often recommend std::shared_ptr and especially
std::unique_ptr as the magical cure for the well-known memory-related
problems of C and 'traditional' C++. However this only replaces one type of
problem (potential memory corruption) with another type of problem
(potentially slow performance), especially in code bases with a high number
of active objects, and a high frequency of object creation and destruction.

The problem is that the memory corruption issues are fixed 'immediately', but
the performance problems typically only show up late in a project. So the
'smart pointer problem' can be a dangerous trap for non-trivial projects,
once the problems rear their ugly head it's usually too late to fix them
because smart pointers have infested the entire code base already.

The problem with smart pointers provided by the C++ standard library are
rooted in their role as 'owner' of a chunk of heap memory. In the default
situation, each object managed by smart pointers is it's own allocation
on the heap. This causes a number of problems:

1. it's not guaranteed that related objects are close to each other in memory, this causes unnessecary cache misses
2. creating and destroying many objects in a short time can be expensive
3. currently, all standard library smart pointers are thread-safe, this makes
the typical single-thread scenario more expensive that it needs to be

Worse, 'object spiderweb' architectures where a big number of objects
reference each other often have a flat 'death by a thousand cuts'
peformance profile, where the code is slow without any obvious performance
hotspots. This can lead to the false impression that the code cannot run any
faster and further performance optimizations are impossible. 

Smart pointers are only really useful in such complicated scenarios with many
objects in complex relationships, their downsides are amplified in exactly
the use case where they are most useful!

The above architecture and performance problems are mostly addressed by a
data-oriented-design philosophy, but it's not obvious what role pointers
(both raw and smart) play in a data-oriented-design code base. Is it best
to use raw pointers, smart pointers, or something else in this case?

The code idea of data-oriented-design is basically to arrange related
'objects' (or let's call them 'data items') close to each other in memory
to avoid cache misses as much as possible when performing 'batch operations'
on those data items.

Storing data items in simple arrays is a natural fit for the data-oriented 
approach, since items in an array are located next to each other in memory.

This means that 'creating' and 'destroying' a single item doesn't involve
allocating and freeing memory. Instead the memory for a whole group of items
is allocated upfront. The whole idea of a pointer as 'owner' of an object
doesn't fit well into the data-oriented-design philosophy, since a single
object or data item simply isn't _important enough_ to justify its own heap
allocation (I'm conveniently ignoring the whole topic of custom allocators
here, they solve the problem of 'one allocation per object', but don't solve
other problems of 'pointers as owners'). As soon as objects are managed in
big numbers, the whole idea of smart pointers to _manage ownership_ pretty
much becomes pointless.

And further following down that path, the whole ideas of raw pointers as
'owners and identifiers' becomes rather pointless too. Once a data item lives
in an array, it makes more sense to identify it through an array index
since this eliminates some dangerous properties of the (more universal)
pointers.

So let's quickly recap before moving on to handles:

- keeping each object in its own heap allocation is bad
- overloading pointers with multiple roles is bad (owners, identifiers, accessors)
- smart pointers fix the most dangerous problems of raw pointers, but are not a good fit for data-oriented-design 

### Handles to the rescue

