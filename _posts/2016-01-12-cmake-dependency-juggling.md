---
layout: post
title: CMake Dependencies Done Right
tags: [cmake]
---

**TL;DR:** if your top-level CMakeLists.txt file is riddled with *add\_definitions()*,
*include\_directories()* and *link\_directories()*, and your
executable's linker-dependency lists are getting too big, this post is for you :)

## The Problem:

CMake suffers a bit from the "draw the rest of the fucking owl" problem: 

Simple projects with a few libraries and executables are easy and
straight-forward to setup, but the docs don't say much how to organize a big project
with dozens of libraries and executables. It requires discipline
and a bit of cmake knowledge to not create a tangled mess.

Typical symptoms are:

- a big and messy root CMakeLists.txt file with many *include\_directories()*,
*link\_directories()* and *add\_definition()* calls 
- executable targets with huge lists of linker dependencies
- on some linkers, unresolved linker errors because the order in which
linker dependencies are given is important

If there's a compile problem like missing header search paths or
preprocessor definitions, the easiest solution is to throw another 
*include\_directories()* or *add_definitions()* into the top-level cmake file,
and this cruft slowly builds up.

The problem with *include\_directories()* and *add\_definitions()* is: they only
propagate downward in the cmake file tree which is usually not useful way down in
a leaf module, because only this one module will know about those include
directories and definitions.

Putting those statements into the root cmake file solves the problems, but is
basically the same as defining a global variable. If a header in my rendering
module needs to find some 3rd party graphics library headers, then every piece
of code that *depends* on my rendering module needs to know the search path,
but not all the other code in my engine.

*Linker dependencies* don't suffer from this 'global variable' problem, but it
is tedious to maintain a flat list of dependencies for many executable targets.
Let's say my rendering module requires OpenGL on Linux, and Direct3D on
Windows. This means all executables need to link against GL on Linux, and
d3d11.lib on Windows. Now the rendering modul is extended to also support Metal
on OSX. If I'm maintaining a flat linker dependency list in the executable
cmake files, I need to touch dozens of cmake files in a complex project to add
the new linker dependencies to all executables.

## The Solution:

CMake provides solutions to all those problems, but they
all depend on defining a proper **dependency tree** for all libraries and
executables with **target\_link\_libraries()**, so that's the first thing 
to get right.

It seems a bit silly to define linker-dependencies for libraries, since
a (static) library is not linked at all. CMake's twist on this is to keep 
the dependency lists of libraries around until an executable is linked, 
and then do a recursive resolve on the whole dependency tree
to get a flat list of link libraries **in the right order** (so it works 
automatically even for old-school linkers where the link-order is important).

Here's an example for our theoretical rendering module, called 'Gfx'. Depending
on the platform, executables using the Gfx module must link against GL, D3D11
or Metal. Also let's say that the Gfx module needs code from two other 
project-internal modules 'Core', and 'IO', these can also be added with
target\_link\_libraries():

{% highlight cmake %}
# in the CMakeLists.txt file of the Gfx module:
add_library(Gfx ${SOURCES})

# Gfx needs project-internal modules Core and IO:
target_link_libraries(Gfx Core IO)

# and executables need to link against the platform's 3D libs:
if (USE_OPENGL)
    target_link_libraries(Gfx GL)
elseif (USE_D3D11)
    target_link_libraries(Gfx d3d11)
elseif (USE_METAL)
    target_link_libraries(Gfx ${metal_framework})
endif()
{% endhighlight %}

If an executable depends on the Gfx module we don't also need to 'manually'
link against Core, IO and the platform's 3D libs, cmake will take care of this,
and because we have defined a proper dependency-tree, cmake will resolve this
tree depth-first, so that the resulting flat list is automatically in the right
order:

{% highlight cmake %}
add_executable(MyGame ${SOURCES})
target_link_libraries(MyGame Gfx)
{% endhighlight %}

So even though only Gfx is given as link-library to the MyGame executable,
it will actually be linked against Gfx, IO, Core and the native 3D libraries.

Doesn't look like a big advantage in this small example, but in a big project
this sort of dependency-hygiene really pays off.

## We need to go Deeper

Now that the linker dependencies have been fixed, that information can be used
to get rid of global preprocessor defines and header search paths. The key is to
use *target_include_directories()* instead of *include_directories()* and
*target_compile_definitions()* instead of *add_definitions()*.

Those new functions don't propagate downward in the cmake file hierarchy, but
upward in the dependency tree, which is much more useful!

Let's say our rendering module from above wants to let other code *which 
depends on the rendering module* (important detail!) know, what
rendering backend is used via a preprocessor define:

{% highlight cmake %}
# in the CMakeLists.txt file of the Gfx module:
add_library(Gfx ${SOURCES})
...
# and executables need to link against the platform's 3D libs:
if (USE_OPENGL)
    target_compile_definitions(Gfx PUBLIC HAS_OPENGL_BACKEND=1)
    ...
elseif (USE_D3D11)
    target_compile_definitions(Gfx PUBLIC HAS_D3D11_BACKEND=1)
    ...
elseif (USE_METAL)
    target_compile_definitions(Gfx PUBLIC HAS_METAL_BACKEND=1)
    ...
endif()
{% endhighlight %}

If the dependencies are set up right, all code which depends on the
Gfx module will now see a HAS\_\*\_BACKEND preprocessor define without
polluting the top-level CMakeLists.txt file.

*target\_include\_directories()* works exactly the same, but for header
search paths. It is interesting though that cmake does **not** offer
a *target\_link\_directories()* function. The motivation behind this seems
to be that helper functions like find\_library() return absolute paths
anyway, so it is preferable to pass an absolute path to
target\_link\_libraries() for external libs.

