---
layout: post
title: Coding Habit Changes
---

A little (fairly random) list of coding habit changes I tried in my weekend
projects (mainly [Oryol](https://github.com/floooh/oryol)) over the past few
years and how they worked out, in no particular order. This is all strictly
IMHO, don't take this as ghospel / the-one-and-only-truth ;)

### What worked well:

- **no setters/getters**: I was used to make class members private and write
  public or protected setter/getter methods. One day I removed all
  setters/getters from Oryol to see 'how it feels', and it just felt right
  (still does). Class declarations in headers are much easier to grok, and code
  is more readable. Const-correctness still works, the only thing you lose is
  the ability to set a breakpoint into the setter/getter methods, but I can
  count on one (maybe two) hands when I ever needed that.

- **no more 'big ideas'**: I stopped long ago believing in 'big architecture
  ideas' that dominate an entire project (Oryol is one result of this, although
  the 'old way' is still visible in places like the Messaging module). Stuff
  like 'everything is an object', 'everything communicates by message passing',
  'all objects have names', 'all objects live in a big tree', everything this,
  everything that... ESPECIALLY if these big ideas are cast into centralized
  code. At one point you inevitably start working around the big idea because
  new things just don't quite fit in anymore. It is better for a project to
  have many 'small, local ideas' and very little shared code. Big ideas can
  still have their place, but only isolated in small leaf-modules where they
  cannot infect an entire project.

- **no inheritance**: I hardly use class inheritance anymore, this is more an
  unintended side effect of other habit changes (mostly because I stopped
  caring about 'designing for code-reuse and extensibility', both are
  well-intended, but dangerous lies).

- **fixed upper bounds, object pools and pre-allocation**: This is hard to put
  into an existing code base but works very well in Oryol. Whenever there need
  to be several things of the same type, Oryol has an upper limit of
  how many of those things can be alive at the same time, either as compile-time
  option or set at application startup. If a limit is reached, the application
  is terminated with an error. Usually those upper bounds exist as a 'rough
  estimate' in the mind of the programmer anyway (most programming solutions
  I've seen work well for some 'intended' number of items, but completely fall
  apart if the number of items is a few order-of-magnitudes higher). Defining
  this upper limit instead of letting everything grow infinitely has (at least)
  2 advantages: 
    - the application can pre-allocate big chunks of memory at startup
      which can simplify memory management greatly
    - more obvious leak detection and debugging, if objects of a specific type
      leak, it is detected early when its pool runs out of free items, instead
      of much later when the process runs out of memory, it's also obvious 
      *what* type of object is leaking

- **no pointers**: This was not really intended but happened as an automatic
  side-effect of keeping objects in fixed-size arrays. There are hardly any
  pointers (of the raw or smart flavour) in Oryol, there are also hardly any
  calls to new/delete/make\_shared/whatever. Instead an 'object handle' is
  usually a simple array index, sometimes with an unique value as 'dangling
  reference protection', or it can be any other type of 'id' that can be mapped
  to one (or more!) memory locations.

- **no tiny heap objects, no pointer-chasing**: ...another side-effect of
  grouping (and processing) objects in flat arrays. There's a lot of C++ code
  out there which creates thousands of tiny objects on the heap, held together
  by smart pointers and long chains of pointer-chasing call like
  get\_this\_obj()-\>get\_that\_obj()-\>do\_this(). This is the typical
  'death-by-a-thousand-cuts' code which is slow without any obvious hotspots
  and thus nearly impossible to optimize. 

- **C++11 range-based for loops**: I started using this when possible, but still
  fallback to the old method quite often for any 'non-trivial' loops

### What kinda worked:

- **upper/lower case to indicate public/private**: I thought Go's 'everything
  that starts with lower case is private' is a neat idea, especially since
  C++'s public/private/friend declaration is quite useless (IMHO). What I want
  from a public/private mechanism is a hint for an API user that a class is not
  for 'public consumption'. I don't even need to enforce this through a compile
  error, just a hint is enough. So I started to use upper case for public
  classes, members and methods, and lower case for 'private' stuff. I think
  this was only partly successful:
    - once 'the dam was broken' I started to use all types of naming schemes, 
      camel case, snake case, etc... especially in small sample projects,
      this looks completely terrible and I need to find a more consistent style 
      again
    - even in Oryol I still mix conventions, in some places, a 'hard' private:
      block makes sense, and in other places all class members are public since
      they need to be accessible from other places in the same modul, 
      and are only 'tagged' as private through the lower case name
- **C++11 auto**: I've started to use 'auto' sparingly if the underlying type
  is either clear from looking at the code, or not at all important. Using auto
  or not can be a subtle hint to someone reading the code. If I want to make a
  point that a type is important in a specific place, I don't use an auto, and
  the other way around.
- **C++11 lambdas (when used wisely)**: When I absolutely need to pass a
  function object with captured arguments I usually do this through a lambda
  now instead of using std::function/std::bind directly (that's just too ugly).
  I wouldn't use this as a 'big idea' for building an entire system around it
  though since std::function objects are quite fat and complex under the hood.

### What I don't want to give up just yet:

- **this->**: I'm still a fan of using 'this' for member access inside class
  methods, otherwise I find it hard to differentiate between local variables
  and member variables (and don't get me started on the m\_ notation)

- **const correctness**: I still think const-correctness is a very useful
  concept, even though I have to admit that I can't think of a case 'where it
  actually prevented a bug' (may be that's exactly the point of using const, I
  don't know). I even started to use const for local variables that should not
  be mutable, but I think that might go a bit too far, especially since I'm not
  always consistent with this. I would actually love if everything in C++ was
  const by default and would have to be explicetly marked as mutable, will
  never happen of course.

### What didn't work:

Only one thing comes to mind, probably because I forgot about stuff that
obviously didn't work pretty fast:

- **inline code inside class declaration**: I tried for a while to move all inline
  code up into the class declaration instead of having it isolated below the
  declaration in 'standalone methods'. I gave up on this except for very tiny
  and unimportant classes because it looks just terrible. I consider the header
  file the most important highlevel documentation of a class, and intermixing
  the only important part with implementation details just looks wrong.  That's also
  a gripe I have with other languages that interleave interface declaration
  with implementation, but I guess I'm in the minority there :)

