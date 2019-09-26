---
layout: post
title: "Modern C" for C++ Peeps
---

When discussing C with C++ programmers I often notice a somewhat outdated
view of C, usually a version of C that's somewhere between C89 and C99,
because that's essentially the "subset of C that's supported by C++".

I can't blame them though because when I started to write C code again I had a
similar outdated view on the language.

Also to be clear, the "Modern C" I'm talking about here is not modern at all,
but already 2 decades old. I'll focus on the subset of C99 which is supported
by clang, gcc and MSCV (clang and gcc is supporting the full C99 standard of course, while (AFAIK) Microsoft's stance unfortunately hasn't changed since this post from 2012: 

https://herbsutter.com/2012/05/03/reader-qa-what-about-vc-and-c99/.

It's not all bad in Visual Studio land though, the Microsoft C compiler
actually supports a pretty solid subset of C99 at least since around 2015 or
so. I guess there are some guerilla warriors at Microsoft which secretly
sneak in some updates to the C compiler when the C++ people are out of office
discussing important stuff in their C++ committee meetings.

Ok, lets get started:

## Modern C is not (a subset of) C++

C++ programmers sometimes recommend to compile C code in 'C++ mode' to take
advantage of the slightly stricter type checking of C++ (more on type safety
below), and that's even recommended in that Microsoft blog post I linked to
above:

> "We recommend that C developers use the C++ compiler to compile C code"

...to be blunt, that's a load of rubbish because C++ only supports a terribly
old-fashioned version of C. The new stuff in C99 which makes C a much
friendlier language isn't supported, at least not in "standard C++". GCC and
especially Clang both have compiler-specific extensions which allow to use
more of C99 in C++ mode, but unfortunately the only compiler where such
non-standard C++ extensions would be really useful (MSVC) doesn't have any of
that.

To iterate on that point, because it's really important:

> C is not a subset of C++

Apart from the fact that C99 code simply doesn't compile in C++ (and probably
never will even in future versions of C++), programming in C requires a
slightly different approach of how to work with data, and how to structure a
project. Trying to 'emulate' missing features from C++ in C usually isn't a
good idea.

C++ is also not a 'replacement' or a 'successor' to C, it's a fork which
slowly created a slightly different dialect of C without much hope that the
two languages can ever be united again (which IMHO is a damn shame, because
being able to mix C with a sane subset of C++ would be really useful for
writing libraries).

But aaaaanyway... it's moot to complain about this from the side lines.

##  The simple stuff

In the unlikely case that you last had a look at C around 1990, a few things
that are allowed in C now are:

- winged comments: ```// this is a comment```
- variables can be declared anywhere, not just at the beginning of a scope block
- the loop variable in a for loop can be declared inside the ```for()``` so
that the variable doesn't leak out into the outer scope: ```for (int i =
...)```
- integer types are much more intuitive: ```uint8_t, uint16_t, int32_t``` etc...
(these are not built-in but defined in the stdint.h header)
- there's now a standardized ```bool``` with ```true and false``` (also not
built in but defined in stdbool.h)

By the way, a subtle yet important difference between C and C++ is a function declaration with an empty argument list:

```c
void my_func() {
    ...
}
```

In C++ this means that the function takes no arguments, but in C this function
accepts *any* number arguments (I can't think of a situation where that would actually 
be useful, since there's no way to access those "variable argument list" inside the
function, I guess it's a leftover from old K&R function declaration syntax).

Instead in C declare the parameter list explicitely as ```void``` so that you
actually get compiler errors when accidently passing arguments to
```my_func```:

```c
void my_func(void) {
    ...
}
```

## Enable all warnings!

C's type system is a bit more relaxed than C++ which can at times be
dangerous. This is also the reason why C++ people sometimes recommend to
compile C code as C++. The more realistic alternative is to enable the
highest warning levels a compiler allows, this generates warnings 
for all cases (I know of) that are errors in C++, and it's a good idea
to do this in C++ too anyway.

These are the relevant flags:

- ```-Wall and -Wextra``` on GCC 
- ```-Weverything``` on Clang
- ```/W4 or /Wall``` on MSVC

Especially /W4 on MSVC will generate a lot of fairly pointless spam though,
so some additional additional *warning hygiene* may be required by 
carefully disabling warnings in selected places. I would advice to not
do this globally though (maybe with a few exceptions), but instead only
suppress specific warnings in specific places after making sure the warnings
are indeed "spam".

The relevant compiler-specific statements are:

```c
#if defined(_MSC_VER)
#pragma warning(push)
#pragma warning(disable:1234)
#elif defined(__clang__)
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wxxx"
#elif defined(__GNUC__)
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wxxx"
#endif

// code which generates uncritical warnings here

#if defined(_MSC_VER)
#pragma warning(pop)
#elif defined(__clang__)
#pragma clang diagnostic pop
#elif defined(__GNUC__)
#pragma GCC diagnostic pop
#endif
```


## Wrap your structs in a typedef

One of the first annoyances a C++ programmer will notice in C is that one needs to write ```struct``` all over the place.

In C++ you can simply do:

```cpp
struct bla_t {
    int a, b, c;
};

bla_t bla = ...;
```

...while in C you must explicitly write ```struct```:

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

This way of typedef'ing from an anonymous struct has a problem though, you 
can't use a forward-declaration of such a typedef from an anonymous struct:

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

Now you have a named struct ```bla_t``` which is typedef'ed to a type alias ```bla_t```.

So if you see a strange typedef like this in C code the reason is that it
enables forward declaration. Useful in library header but otherwise you
don't need to care about this.


## Use struct wrappers for strong typing

This isn't really restricted to C, since C++ suffers from the same problem:

> typedef doesn't define a type, only a "weak" type alias

Typedef is only a weak *type alias* not a new type, meaning there's no
warning when assigning a different type from the same base type:

```c
typedef int meters_t;
typedef int hours_t;

meters_t m = 1;
hours_t h = m;    // this isn't an error, but should
```

Wrapping the types into a struct makes this code properly typesafe:

```c
typedef struct { int val; } meters_t;
typedef struct { int val; } hours_t;

meters_t m = { 1 };
kilometers_t km = m;    // compile error!
```

## Initialization in C99

C99's new initialization features are by far the biggest usability
improvement over C89 to a point where it almost feels like a new language,
and it makes the many different ways C++ offers for initialization look a bit
silly.

The two relevant features are ```compound literals``` and ```designated initialization```.

Both together let you do things like this:

```c
typedef struct { float x, y; } vec2;

vec2 v0 = { 1.0f, 2.0f };
vec2 v1 = { .x = 1.0f, .y = 2.0f };
vec2 v2 = { .y = 2.0f };    // missing members are set to zero
```

For initializing globals, only compile-time variables are allowed
(which is a good thing because it completely avoids C++'s undefined
initialization order problem).

Inside functions, variable values can be used for initialization:

```c
float get_x(void) {
    return 1.0f;
}

void bla(void) {
    vec2 v0 = { .x = get_x(), .y = 2.0f };
}
```

Unfortunately the C compiler can't always infer the type from the left side
of an assignment:

```c
    vec2 v0;
    // this doesn't work
    v0 = { 1.0f, 2.0f };
    // instead a type hint is needed:
    v0 = (vec2) { 1.0f, 2.0f };
```

Here's a more interesting real-world example from the
[sokol_gfx.h](https://github.com/floooh/sokol) example code. This initializes an
'option bag' structure for creating a Pipeline State Object:

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
the bad news is that it's only a limited subset of the C99 feature. For
instance this won't work in C++20:

```c
    ...
    .buffers[0].stride = 28,
    ...
```

You can use an array index in C++20, and you can't chain designators like that.

## Don't be afraid to pass and return structs by value

There's still a lot of outdated 'optimization advice' about passing and
returning structs by value. In C++ this is often even justified because
copying a non-trivial C++ object may be more expensive than it looks from the
outside.

The situation in C is a whole lot simpler, copying a struct is always a
simple operation without involving 'custom code'.

Furthermore, the "new" 64-bit calling conventions pack small structs into
registers (at least on Intel, don't know what's the situation on ARM). So
that there's a good chance that passing small-ish structs by value doesn't
even hit memory.

And on top of that, with optimization enabled, and the compiler being
able to inline a function call, even bigger structs are most likely
'optimized away'.

Here's an example how "old school code" might look like to add two simple
structs:

```c
struct float2 { float x, y; };

void addf2(const struct float2* v0, const struct float2* v1, struct float2* out) {
    out->x = v0->x + v1->y;
    out->y = v0->y + v1->y;
}

...
    struct float2 v0, v1, v2;
    v0.x = 1.0f; v0.y = 2.0f;
    v1.x = 3.0f; v1.y = 4.0f;
    addf2(&v0, &v1, &v2);
...
```

Here's the 'modern' version:

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
typedef struct { float x, y; } float2;

float2 addf2(float2 v0, float2 v1) {
    return (float2) { v0.x + v1.x, v0.y + v1.y };
}

...
    float2 v3 = addf2((float2){ 1.0f, 2.0f }, (float2){ 3.0f, 4.0f });
...
}
```




=== Use struct arguments with designated init for "option bags"

=== Be afraid of pointers!

=== But pointers as "borrow reference function args" are ok

    void func(const bla_t* bla);

    func(&(bla_t*) { ... });

=== What to do about missing RAII

- embrace the idea of "plain old data" structs, it assures that copying a
  struct always a straight copy and destruction is a no-op
- don't have bits "dangling off" your structs which need special
  attention for copying and destruction
- use nested structs and arrays to place many things into one
- don't use "owning pointers" which need some sort of lifetime tracking,
  instead use "tagged handles" with dangling protection
- try to eliminate malloc/free from "high level code"
- try to eliminate fined-grained malloc/free entirely
- strongly prefer the stack over the heap
- if complex construction are absolute needed, create helper functions which return a new item

    bla_t bla0 = make_bla(...);
    bla_t bla1 = make_bla((bla_desc_t) { ... });
    
- alternatively offer slightly less convenient but more flexible init functions

    bla_t bla0;
    init_bla(&bla0, (bla_desc_t) { ... })

=== Don't think in "classes", instead think in "modules"

=== Don't think in "objects", instead think in "arrays"

=== Should everybody switch back to C now?

...
