---
layout: post
title: Modern C for C++ Peeps
---

(update 28-Sep-2019: some fixes and error corrections)

When discussing C with C++ programmers I often notice a somewhat outdated
view of C, usually a familiarity with a C dialect that lies somewhere between
C89 and C99, because that's essentially the "subset of C that's supported by
C++".

I can't blame them though because when I started writing C code again I had
a similar outdated view of the language.

Also to be clear, the 'Modern C' I'm talking about here is not modern at all,
but already two decades old. I'll focus on the subset of C99 which is
supported by clang, gcc and MSVC (clang and gcc both fully support the latest
C standards of course, while (AFAIK) Microsoft's stance unfortunately hasn't
changed much since [this post from 2012](https://herbsutter.com/2012/05/03/reader-qa-what-about-vc-and-c99/).

It's not all bad in Visual Studio land though, the Microsoft C compiler
actually supports a pretty solid subset of C99 since around VS2015. I guess
there are some guerilla warriors at Microsoft who secretly sneak updates
into the C compiler when the C++ people are out in the field on one of their C++
Committee trips.

But lets get started:

## Modern C is not a subset of C++

C++ programmers sometimes recommend to compile C code in 'C++ mode' to take
advantage of the slightly stricter type checking in C++ (more on type safety
below), and that's even recommended in that Microsoft blog post I linked to
above:

> "We recommend that C developers use the C++ compiler to compile C code"

...I'm sorry to be a bit blunt, but that's a load of rubbish because C++
*still* only supports a terribly old-fashioned version of C. The new stuff in
C99 which makes C a much friendlier language isn't supported in C++, at least
not in "standard C++".

GCC and especially Clang both have compiler-specific extensions which allow
to use more of C99 in C++ mode, but unfortunately the only compiler where
such non-standard C++ extensions would be really useful (MSVC) doesn't have
any.

To iterate on that point, because it's really important:

> C is not a subset of C++

Apart from the fact that C99 code simply doesn't compile in C++ (and probably
never will even in future versions of C++), programming in C requires a
different approach of how to work with data, and sometimes how to structure a
project (for instance: don't think in classes, but in modules and systems).
Trying to 'emulate' missing features from C++ in C usually isn't such a great
idea.

C++ is also not a 'replacement' or a 'successor' to C, it's a fork which
slowly 'devolved' its C subset into a slightly different dialect of C
without much hope that the two languages can ever be united again (which IMHO
is a damn shame, because being able to mix C with a sane subset of C++ would
be really useful for writing libraries).

But aaaaanyway... it's moot to complain about this from the side lines.

##  The simple stuff

In the unlikely case that you last had a look at C around 1990, lets get the
simple things out of the way first:

- winged comments (```//```) are allowed
- variables can be declared anywhere, not just at the beginning of a scope block
- the loop variable in a for loop can be declared inside the ```for()``` so
that the variable doesn't leak into the outer scope: ```for (int i =
...)```
- integer types have much clearer names now: ```uint8_t, uint16_t, int32_t``` etc...
(these are not built-in but defined in the stdint.h header)
- there's now a standardized ```bool``` with ```true/false``` (also not
built-in but defined in stdbool.h) (**update:** I made a small error here,
the bool type is indeed a (private) builtin type, usually named something
like _Bool, the stdbool.h header just redefines this to the common 'bool'
name)

By the way, a subtle yet important difference between C and C++ is a function
declaration with an empty argument list:

```c
void my_func() {
    ...
}
```

In C++ this function takes no arguments, but in C this function takes *any*
number of arguments (I can't think of a situation where that would actually be
useful, since there's no way to access this "variable argument list" inside
the function, I guess it's a leftover 'syntax pollution' from old K&R style
function declaration syntax).

Instead in C, declare the parameter list explicitely as 'void' so that you
actually get compiler errors when accidently passing arguments to 'my_func()':

```c
void my_func(void) {
    ...
}
```

## Enable all warnings!

C's type system is in some details a bit more relaxed than C++ which can at
times be annoying. This is also the reason why C++ people sometimes
recommend to compile C code as C++. The more realistic alternative is to
enable the highest warning levels a compiler allows. This generates warnings
for all type system related problems (I know of) which C allows but C++ catches
as errors.

These are the relevant flags:

- ```-Wall and -Wextra``` on GCC 
- ```-Weverything``` on Clang
- ```/W4 or /Wall``` on MSVC

Especially /W4 on MSVC will generate a lot of fairly pointless spam though,
so some additional *warning hygiene* may be required by 
carefully disabling warnings in selected places. I would advice to not
do this globally though (ok, maybe with a few exceptions), but instead only
suppress specific warnings in specific places after making sure the warnings
are indeed uncritical spam.

Proper warning hygiene is a topic worth of its own blog post though.

## Wrap your structs in a typedef

One of the first annoyances a C++ programmer will notice in C is that one needs to write ```struct``` all over the place.

In C++ you can simply do:

```cpp
struct bla_t {
    int a, b, c;
};

bla_t bla = ...;
```

...while in C you must explicitly write:

```c
struct bla_t bla = ...;
```

Wrapping a struct in a typedef fixes that, and that's why you'll see this a lot in C code:

```c
typedef struct {
    int a, b, c;
} bla_t;

bla_t bla = ...;
```

This way of typedef'ing from an 'adhoc' anonymous struct has a problem though, you 
can't forward-declare the struct:

```c
// forward-declaring bla_t and a function using bla_t:
struct bla_t;
void func(struct bla_t bla);

// actual struct and function, using the typedef:
typedef struct {
    int a, b, c;
} bla_t;

void func(bla_t bla) { // <= warning 'parameter different from declaration'
    ...
}
```

The workaround is to rewrite the typedef like this:

```c
typedef struct bla_t { int a, b, c; } bla_t;
```

Now you have a named struct ```bla_t``` which is typedef'ed to a type alias ```bla_t```,
and the named struct can be properly forward-declared.

So if you see a strange 'redundant' typedef like this in C code the reason is that it
enables forward declaration. 

**updates**: (1) apparently the Linux coding guidelines discourage typedef'ing
a struct, and (2) the POSIX standard reserves the '_t' postfix for its own typenames
to prevent collisions with user types -- make of that what you will ;)

## Use struct wrappers for strong typing

Typedef's 'weakness' is another annoyance, both in C and C++:

> **typedef** - creates an alias that can be used anywhere in place of a (possibly complex) type name.

Typedef only creates a weak *type alias* not a proper new type (it's really not
much better than a preprocessor define), meaning there's no warning when
assigning to a different type from the same base type:

```c
typedef int meters_t;
typedef int hours_t;

meters_t m = 1;
hours_t h = m;    // this isn't an error, but it really should be
```

Wrapping the types into a struct makes this code properly typesafe:

```c
typedef struct { int val; } meters_t;
typedef struct { int val; } hours_t;

meters_t m = { 1 };
hours_t h = m;    // compile error!
```

Depending on how much of a fan of strong typing you are, this approach makes
sense both in C and C++.

## Initialization in C99

C99's new initialization features are by far the biggest usability
improvement over C89 to a point where it almost feels like a new language,
(and to be honest, it makes the many different ways C++ offers for
initialization look a bit silly).

The two relevant features are ```compound literals``` and ```designated initialization```.

Both together let you do things like this:

```c
typedef struct { float x, y; } vec2;

vec2 v0 = { 1.0f, 2.0f };
vec2 v1 = { .x = 1.0f, .y = 2.0f };
vec2 v2 = { .y = 2.0f };    // missing struct members are set to zero
```

For globals, only compile-time constants are allowed for initialization
(which is a good thing because it completely avoids C++'s undefined
initialization order problem for globals).

Inside functions, runtime-variable values can be used for initialization:

```c
float get_x(void) {
    return 1.0f;
}

void bla(void) {
    vec2 v0 = { .x = get_x(), .y = 2.0f };
}
```

Unfortunately the C compiler can't always infer the type from the left-hand side
of an assignment:

```c
    vec2 v0;
    // this doesn't work
    v0 = { 1.0f, 2.0f };
    // instead a type hint is needed:
    v0 = (vec2) { 1.0f, 2.0f };
```

Here's a more interesting real-world example from the
[sokol_gfx.h](https://github.com/floooh/sokol) example code. This initializes a
nested 'option bag' structure which contains dozens of members, where only a few
of the members get non-default values:

```c
sg_pipeline_desc pip_desc = {
    .layout = {
        .buffers[0].stride = 28,
        .attrs = {
            [ATTR_vs_position].format = SG_VERTEXFORMAT_FLOAT3,
            [ATTR_vs_color0].format   = SG_VERTEXFORMAT_FLOAT4
        }
    },
    .shader = shd,
    .index_type = SG_INDEXTYPE_UINT16,
    .depth_stencil = {
        .depth_compare_func = SG_COMPAREFUNC_LESS_EQUAL,
        .depth_write_enabled = true,
    },
    .rasterizer.cull_mode = SG_CULLMODE_BACK,
    .rasterizer.sample_count = SAMPLE_COUNT,
    .label = "cube-pipeline"
};
```

The good news is that C++20 is getting basic designated initialization too,
the bad news is that it will only be a very limited subset of the C99 feature. For
instance this won't work in C++20:

```c
    ...
    .buffers[0].stride = 28,
    ...
```

You can't use an array index in C++20, and you can't chain designators like that.

(worth mentioning that at least Clang has a C++ extension which allows much
more powerful C99-style designated-initialization already now than what C++20
will offer, such code is not portable to other compilers though)

## Don't be afraid to pass and return structs by value

There's still a lot of outdated 'optimization advice' about passing and
returning structs by value in C and C++ around. In C++ this advice is
sometimes even justified because copying a non-trivial C++ object may be much
more expensive than it looks on the surface because complex custom copying
code might be invoked under the hood (passing std::string objects by value is
the best/worst example).

The situation in C is a whole lot simpler, copying a struct is always a
straight data copy without involving custom code.

Furthermore, the "new" 64-bit calling conventions pack small structs into
registers (at least on Intel, don't know what's the situation on ARM). So
there's a good chance that passing small-ish structs by value doesn't
ever hit memory.

And on top of that, with optimization enabled, and the compiler being able to
inline a function call, even bigger structs are most likely 'optimized away'
completely. But don't take my word for granted! When in doubt, always check
[godbolt.org](https://www.godbolt.org/) and/or the code generated by your compiler.

Here's an example how "old-fashioned C code" might look like to add two "2D vectors":

```c
struct float2 { float x, y; };

void addf2(const struct float2* v0, const struct float2* v1, struct float2* out) {
    out->x = v0->x + v1->x;
    out->y = v0->y + v1->y;
}

...
    struct float2 v0, v1, v2;
    v0.x = 1.0f; v0.y = 2.0f;
    v1.x = 3.0f; v1.y = 4.0f;
    addf2(&v0, &v1, &v2);
...
```

Here's the 'Modern C' version:

```c
typedef struct { float x, y; } float2;

float2 addf2(float2 v0, float2 v1) {
    return (float2) { v0.x + v1.x, v0.y + v1.y };
}

...
    float2 v0 = { 1.0f, 2.0f };
    float2 v1 = { 3.0f, 4.0f };
    float2 v3 = addf2(v0, v1);
...
```

You can also move the initialization of the two inputs right into the function call:

```c
    float2 v3 = addf2((float2){ 1.0f, 2.0f }, (float2){ 3.0f, 4.0f });
```

## Named optional arguments

C99's designated initialization enables an interesting 'Easter Egg' feature:
let's say you have a C function which requires many input parameters, most
of them optional (in which case default values should be used). In C++
you can have optional args with default values, but they must appear in
order and at the end of the argument list.

C99's designated initialization to the rescue. Put the argument list
into a 'parameter struct', like in this function from
[sokol_gfx.h](https://github.com/floooh/sokol/blob/b5e43ec43eb3d0a8a73eadd5db979291be3e039d/sokol_gfx.h#L2006):

```c
sg_image sg_make_image(const sg_image_desc* desc);
```

This takes a pointer to a big "option bag" struct with creation parameters
for an image object, where the parameters have "useful defaults". In C99 you
can call the function while setting up the option-bag struct
right in the function call:

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 256,
    .height = 256
});
```

(note how it's possible in C99 to take the address of a temporary inside
a function call, this is an example of code that's completely valid in
C, but not in C++)

This function call creates an image of size 256x256 with all other parameters
using their default values.

In case I also need to use a non-default pixel format I simply add that to
the option-bag parameter list, the parameter order doesn't matter:

```c
sg_image img = sg_make_image(&(sg_image_desc){
    .width = 256,
    .height = 256,
    .pixel_format = SG_PIXELFORMAT_R8
});
```

Etc etc... Of course it's possible to do something similar in C++ with an
```sg::image_desc``` class and the builder pattern, but this requires to
write a lot of rather boring boilerplate code to implement the builder
pattern on the class author's side.

## Be (somewhat) afraid of pointers

IMHO, pointers in C should be treated like the ```unsafe``` keyword in Rust. The
presence of pointers in C code and in structs always requires special
attention and mental effort to read and understand all the code 'tainted' by
those pointers.

To be a bit more specific **owning pointers** are the main problem, meaning
pointers which own the thing they point to, and where the pointer might
outlive the pointee, or the pointee might move to a different memory location
(familiar to C++ programmers as everybody's favourite memory corruption feature
AKA 'iterator invalidation').

Pointers which are 'immutable borrow references' are usually ok as 
function arguments though, the important point is that the pointed
to object is only 'borrowed' for the duration of the function call,
and no ownership transfer takes place.

But as you can see, C pointers always come with their own set of caveats and
a lot of 'explanation overhead' for each use of a pointer. Thus it's best to
avoid them alltogether (or at least "as much as possible").

Now I can literally hear the audience burst into laughter crying
"C without pointers? How's that gonna work, smart-ass?!?". 

I'll get straight to that in the next section:

## What to do about missing RAII

I'm using the term RAII (which IMHO is one of the worst names in computing)
for C++'s ability to automatically call user-defined destruction code at the
end of a scope block, and when copying or moving objects (these are the actually
important parts of RAII, not the 'fused' allocation and initialization of an 
object).

My opinion (and of course you don't have to agree with it) on RAII is that it
is mostly useful in the same situations where garbage collection is useful:
automatic memory management - more specifically keeping track of myriads of
tiny memory allocations and deciding when it's safe to free them.

Yeah I know I knooow... theoretically RAII is about *general resource
management*, not just about memory. But at least in my experience, it's
always about memory management.

Garbage collection or RAII are certainly great features to have - **assuming**
having tons of small memory allocations to keep track of is nothing to worry
about.

And that's the core of the problem right there. If you don't want to worry
about memory management it will ineviatably come back to haunt you when it's
too late to do anything about it. It doesn't matter whether *many small
allocations* are managed through a GC or through RAII. The problem is the
*many small allocations*.

If you don't have such small (and often hidden) memory allocations happening
decentralized all over the code in the first place, both GC and RAII lose
most of their appeal. Controversial claim, I know, and I realize that it's very
easy to make such a claim without having a million-line C code base maintained
by a huge team under the belt to back it up (but at least I have the
counter-experience of a million-line C++ OOP code base which does its memory
management through smart pointers - which admittedly is very robust, but
also slow and impossible to meaningfully profile and optimize).

...aaanyhoo... back to "what to do about the missing RAII":

The smart-ass advice is of course "don't do many small memory allocations".

A more reasonable advice is to work with what C offers, instead of trying
to work around it.

The 'C way' is to have 'dumb structs' instead of 'smart objects'. A struct is
just some plain data blob without a behaviour of its own. When the data is copied,
it's always a simple copy. When the data is destroyed, it's always a no-op.

Don't build data structures with 'stuff dangling off' which needs to be tracked
(for instance pointers to a unique memory allocation).

Don't build data structures which require special 'deep copy' operations.

Don't waste a unique allocation for a single data item and don't allocate
in random places in your code, instead keep many data items of the same type
stored in few arrays managed by central 'systems' and reference them
through [tagged index handles](https://floooh.github.io/2018/06/17/handles-vs-pointers.html).

Consider keeping small transient data structures on the stack and pass them around
by value instead of wasting a heap allocation on them.

Embrace DOD (Data Oriented Design), this is built around the basic idea that
data items never come alone and should be stored and processed in
simple linear arrays to improve CPU cache hit rate. The same core idea is
also useful for drastically reducing the number and frequency of memory
allocations, and getting rid of 'owning pointers'.

And you don't need a fancy high-level language for all of that.

## So... should I switch to C now or what?

Erm no, that's not the intention behind this blog post, at most doing my tiny
part to update the somewhat prevalent view (in *some* circles at least) of C
as an outdated language which must be replaced at all cost (usually with
languages that don't quite understand the essence of why people actually
choose C).

I don't want to fuel the language wars, and higher level general languages
like Rust, C# or C++ of course have their place in the world, but so have
small languages like C. 

Because sometimes it simply doesn't make sense to bring an aircraft carrier to a
knife fight ;)
