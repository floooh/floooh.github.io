---
layout: post
title: "Oryol: System Design Philosophy"
---

I think I have written (and re-written, and re-re-written) enough different
Oryol modules now that a pattern has emerged to a point where I feel comfortable writing a blog post.

Please do *not* take this post as an attempt to sell you on design patterns though.
IMHO upfront software design is overrated, continuous refinement 
is what counts :)

### Class Collections vs System Modules

Oryol seems to have arrived at two different module types, one is just
a collection of related helper classes. The [Resource module](https://github.com/floooh/oryol/tree/master/code/Modules/Resource)
or [Asset module](https://github.com/floooh/oryol/tree/master/code/Modules/Assets)
are typical examples.

And then there are the **system** modules, which have a central interface,
are initialized at application start, have internal state and provide some
service to the application code like rendering, input or loading data.
Typical system modules are
[IO](https://github.com/floooh/oryol/tree/master/code/Modules/IO),
[Gfx](https://github.com/floooh/oryol/tree/master/code/Modules/Gfx) or
[Input](https://github.com/floooh/oryol/tree/master/code/Modules/Input).

This post is mainly about the design of **system modules**.

### Typical Properties of Oryol Code

Code written against Oryol has a few interesting properties:

- you never need to call *new*, *make_shared* or similar heap-allocation functions
- you never need to care about ownership, there are no smart pointers, and
there is no concept of 'ownership transfer'
- there are (nearly) no derivable classes and virtual-method-interfaces

There is a minor exception though, Oryol currently *does* have 3 classes with
virtual methods which are user-deriveable, are created on the heap and
lifetime-managed by smart pointers:

- **FileSystemBase**: for user-provided filesystem implementations
- **MeshLoaderBase**: for user-provided mesh resource loaders
- **TextureLoaderBase**: the same for textures

In those few cases where Oryol provides such an 'extension interface' I decided
to go with the standard C++ way instead of inventing something 
new. I'm not entirely happy with all 3 of those classes though (for various
minor reasons), so may be they will disappear completely one day. In the big
picture, these 3 classes also only play a minor role, they are not
used in 'typical' Oryol code.

### The Public System Interface

A system module usually has a single public interface in a single header,
similar to what a C library would provide. There is no public
'system object', the interface class consists of static methods only.

For instance: The **Gfx** module has the header **"Gfx/Gfx.h"** which
exposes static methods like **Gfx::ApplyDrawState()**, **Gfx::Draw()** etc.

All interaction with the system goes through this central 
interface, which also means that you only need to look at this
one header to learn how a system works.

If a system needs any public types besides the central interface those
types are usually grouped in another single header, for instance
[Input/InputTypes.h](https://github.com/floooh/oryol/blob/master/code/Modules/Input/InputTypes.h). The main reason why those
types are in a separate header is that they must also be known to the
private areas of a module while the system-private code will never call
into the public interface API.

The public types are almost always plain-old-data structs
or at most simple struct-like classes with a few 'uninteresting'
helper methods.

All in all, a system module's public interface is much closer to
what a C library would provide, and unlike many 'traditional' C++
frameworks.

### Public vs Private Classes

In general the concept of 'privacy' in Oryol only exists as a hint to the
programmer and for 'code hygiene', it is not enforced by the compiler.

Usually the separation line between public and private area doesn't go
through the middle of a class (there are hardly any C++ *private:* keywords
in the code), instead entire classes of a system module either belong to the
public interface of the module, or are considered module-private.

The hint to the programmer is that public types and functions start with
uppercase letters, private types and functions with lowercase letters.
The compiler will not prevent the programmer from accessing private types,
but this is 'here be dragons' territory. The private interfaces can change at
any time without warning, while the public types hopefully don't need to
change that often.

All private code of a module usually lives in a subdirectory called
'private', this makes it easy to identify the public and private areas while
browsing the code, and it helps mentally to separate public from
private types when writing new code.

The 'code hygiene' part is mostly about header inclusion, private or
operating system headers like Windows.h should not leak into application
code, which helps with compile times, and annoyances like Windows.h adding
global defines for min() and max().

For instance if you look at the [Gfx.h header](https://github.com/floooh/oryol/blob/master/code/Modules/Gfx/Gfx.h) 
you'll notice that it doesn't include any headers from the private area
(most notably, no Windows.h, D3D11.h, GL or Metal headers), instead 
these are only included in the [Gfx.cc implementation file](https://github.com/floooh/oryol/blob/master/code/Modules/Gfx/Gfx.cc).

### Separation of platform-specific code

Platform specific code lives in the private area of a module,
usually in their own directories (for instance see here
in the [Gfx module](https://github.com/floooh/oryol/tree/master/code/Modules/Gfx/private)
for an example how the rendering backend code for different 3D APIs 
is separated).

Oryol uses 'compile time polymorphism' to hide platform specific code under
generic class names, for instance the platform specific 3D API renderers are
resolved into a generic ['renderer'
class](https://github.com/floooh/oryol/blob/master/code/Modules/Gfx/private/renderer.h).
The renderer class is not directly exposed to the application code though,
instead the Gfx interface forwards calls into the renderer. But since there
is no dynamic dispatch (virtual method calls) happening anywhere, and Oryol
applications are fully statically linked, the compiler may actually decide to
inline platform-specific code into the call site, the separation between
platform-agnostic and platform-specific code only exists on the source code
level.

### Object Creation and Lifetime Management

Some system modules need to create and destroy objects under the control
of the application code (for instance graphics resources like textures
or meshes).

It is interesting that most Oryol modules
*don't* need such object management though. For instance among the Core Oryol
modules, only the Gfx module needs to manage objects, all other modules
don't. This is mostly a side-effect of trying to design interfaces that look
more like C libraries, and not "OOP frameworks".

If an Oryol module needs to create objects under control of the application
code, it will completely handle the actual creation and destruction 
internally, and it will always be the sole owner of the object.
This means that application code will never call *new*, *make_shared*
or similar, instead it will call a creation function on the module
interface.

The result of the creation function is an opaque handle
(in Oryol this is usually called an **Id**), it will *never* be 
a C++ object pointer (neither of the smart or raw kind), and
in some (most?) systems, the application code never needs to
access the underlying C++ object directly.

By handing out Ids instead of pointers a lot of typical C/C++ memory
management issues are avoided. The application code cannot accidently access
an object that no longer exists, and it doesn't need to care whether an
object is copyable, moveable, etc...

There are some systems where an application can gain direct access to the
underlying C++ object, this is usually a compromise to avoid a too complex
interface API, the only rule is that access is only 'borrowed', the
application code should never store a pointer to the underlying object, 
and it must be aware that the memory location of an object might change 
if the internal state of the system changes.

Here is an example of such a direct object access from the new
oryol-animation module to get the number of bones of a skeleton:

```cpp
// skeletonId is the opaque id of a character skeleton object
// get the number of bones in the skeleton:
int numBones = Anim::Skeleton(skeletonId).NumBones;
```

How is a dangling access prevented in this case?

The answer is in the Id object. Usually this is an integer divided
into 3 bit-groups:

- an index into an internal object array
- a few type bits (for instance mesh vs texture vs drawstate)
- a 'unique tag', this is a counter that's incremented which each
new Id that is handed out

To convert an Id into an object pointer, the index is used for an array
access into an internal object pool, if this slot is 'unoccupied', the object
no longer exists. But it can also happen that the slot has been re-occupied
by a new object. This is what the 'unique tag' is used for. The new object in
the slot will have a different tag, and thus a dangling access can be
detected.

What happens on such a dangling access is up to the system. It could
'crash' with an assert, it could return a 'dummy object' in its 
default state, or an operation involving the dangling object
could be silently skipped or produce a warning message. What will
*not* happen is a segfault, or scribbling over random memory.

Of course converting an Id to a pointer is more expensive than a direct
pointer access even though it's only an array access and integer
comparision. The point here is that the system interface should be designed
in a way that such high-frequency accesses are not necessary in the
first place.

### Memory Management Simplifications

Since system modules completely control the entire lifecycle of objects, they
also have a lot of freedom to simplify memory management:

- the system can keep objects in pre-allocated arrays to minimize allocations 
(for instance, a typical Oryol frame has no allocations at all)
- systems don't need to 'force' concepts into objects that don't naturally 
fit into objects, it is easier for the system to arrange its internal
data in any way it wants
- there's usually no need for the standard C++ class boilerplate code for
construction, copy- and move-assignment, etc..)

With memory allocation being very infrequent, a few related 
problems also simply disappear:

- there's no need for a fast general allocator (like jemalloc), instead
it may make sense to choose a slow-but-small allocator to reduce
binary size (for instance on asm.js/wasm where the allocator is linked
into the binary).
- there's also no need to invent C++ custom allocators to
workaround problems with the standard allocator
- debugging memory-related problems becomes a whole lot easier
since there's much less 'noise' to filter out

I also discovered that it is much easier to write bindings for other
languages for such 'C style modules' since there's no need to translate the
concept of complex C++ classes across language borders. Only the central
system interface and the object ids (which are simple integers) need to be
brought over. This eliminates the need for complex translation tools which
try to convert C++ class declarations to other languages.

Another interesting thought is that if Oryol would be built in a 
garbage-collected language it would most likely arrive at a very 
similar design to minimize the amount of 'garbage' that's produced
per frame.

### The End

And that's it for today. 

Recently I tried out a few interesting things about
'array-based memory-management' in the new Oryol animation module,
so may be that's the topic for the next blog post.

The realization was basically that the main purpose of container
classes is not 'iterating over items' but 
'simplifying memory management'. I haven't thought too deeply about the
implications for the Oryol container classes yet though, so the idea
needs a bit more time to roll around in the back of my head :)

Until next time!
