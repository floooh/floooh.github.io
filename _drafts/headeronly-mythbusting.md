---
layout: post
title: "Header-only myth-busting"
---

**TL;DR**: how source file organization in C and C++ projects affects compile times.

Whenever "header-only libraries" are discussed, someone inevitably points out
that nobody should ever use them because they drastically increase
compilation time in non-trivial projects. Then people counter that "header-only
libs" can actually improve compilation time, even in big projects.

Both are right because both sides mean different flavours of "header-only",
and quite a many people (especially from the C++ world) are not even aware
that different flavours exist.

So lets get the most important confusion out of the way first:

## "Header-only" vs "single-file" libraries

Names usually make no sense in computing, and the naming situation around
header-only libraries is no exception. By convention, different names have 
emerged for the different flavours of "libraries made of headers only", and for the sake
of clarity I'll use those (somewhat) established names in this blog post, even though
I don't agree with them. IMHO *all* libraries only made of headers are "header-only",
but let's stick to the convention for this blog post.

Before looking at the different flavours of header-only libs I need to
establish another convention though:

### Declaration vs Implementation

I'm often talking about the **declaration part** and **implementation part**
of a library in this blog post. This is what I mean with those:

The **declaration part** of a library consists of all the public data
structures and function signatures which a *user* of the library
needs to know in order to use the library. The declaration part is identical
with the *public interface* of the library.

The **implementation part** is basically "everything else". All the code,
and private data structures which *implement* the library's functionality
but don't need to be exposed to the user of the library.

In "traditional" C89 the relationship between the declaration and
implementation was simple: The declaration part of a library goes into a
header file which is included in each source file the library is used, and the
implementation part goes into a source file which is only compiled once.

If the implementation code would simply be merged with the declaration into
the same header in C89, the header could only be used once in the whole
project, otherwise each "compilation unit" including the header would contain
the implementation code of that library and the linker would get confused.

In C++ the clear separation between a declaration and implementation got
all muddled up through the *inline* keyword and more importantly the way how
*templates* are implemented. Suddenly implementation details had to leak over
to the "user side". C++, templates and the STL will get their own section
later, for now let's get back to the naming convention for different flavours
of "libraries which are made of headers only":

### Header-only libraries

The name **header-only library** describes a library which only consists of
one or more header files and where there's no clear separation between the
declaration and implementation parts of the library. Instead data structures
which are only need by the implementation are visible to the includer, and
all implementation code is either implemented as template code or marked as
inline. A lot (most?? all???) C++ header-only libraries are implemented like
this.

C has (unfortunately) adopted the *inline* keyword since C99 as well, so it's
possible to write such header-only libs in C as well.

This is the "bad" flavour of "libraries made of headers only".

### Single-file libraries (aka STB-style)

TODO

## Measuring compile times

TODO

## C++, STL and the "one file per class rule"

TODO

## How to compile faster with STB-style libs

TODO

## More STB-style goodies

TODO: config defines, extension headers, ...








