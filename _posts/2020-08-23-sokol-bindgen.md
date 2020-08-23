---
layout: post
title: "Automatic Language Bindings"
---

I got caught up in a nice little rabbit-hole in the past few weeks after
discovering a clang feature that I wasn't aware of before: Automatically
generating language bindings for the Sokol headers from an AST (Abstract
Syntax Tree) dumped via clang. I've created a little proof-of-concept with
Zig as target language (which admittedly is a bit of an odd choice of
language to generate bindings for, because Zig can import C headers directly,
more on the reasoning behind this further down in this post).

## Backstory

One of the reasons why I switched to C for writing libraries was
that I wanted those libraries to be useful outside of the C/C++ world. 

Having a C-API is the important part of this idea, because C-APIs are the
defacto 'lingua franca' which allows programming languages of all kinds to
talk to each other.

The underlying implementation language isn't all that important, but writing
the implementation in C too makes sense because it prevents getting tangled
up in fancy high-level-language concepts which then in turn might be hard to
express in a simple C-API.

But having a C-API isn't quite enough. Some C-APIs are better suited for
language interoperability than others, I only discovered those subtle details
while working on the Sokol headers, and I'm still discovering new ones while
experimenting with language bindings. Unfortunately some features which make
an API more suitable for language interoperability may also make the API less
convenient to use from C. It may be difficult to find the right balance 
there.

An example:

I had been working with OpenGL for decades before I noticed an interesting
property of the OpenGL API:

OpenGL has no data structures. None at all.

The entire OpenGL API is made of functions, taking simple value-arguments, or
at most pointers to C-strings or binary-blobs, but never pointers to data
structures. This means one can create OpenGL language bindings without having
to deal with how data structures are layed out in memory, which is especially
useful for very high-level languages which sometimes don't even allow this level
of control over memory layout.

The downside of this is that OpenGL is a very verbose API. Such a function
API can either have many functions with few arguments, or few functions
with many arguments.

Other 3D-APIs like D3D11 or Metal group related data items into data structures,
which reduces API complexity via encapsulation (I'm conveniently ignoring Vulkan
and D3D12 as they don't really support my argument that a data-structure-API
is inherently simple ;) )

For language bindings, data structures can be problematic. The involved
languages must agree on how the content of the data structure is layed out
in memory, with all the alignment and padding rules that come with it. And
every change to the source structure, like changing the type or order of
struct field must be tracked precisely in the language binding, otherwise
hard to debug errors will result.

## Manually maintained language bindings are problematic

Ideally there would be language bindings for the Sokol headers for as many
languages as possible. But if those language bindings need to be manually
maintained this poses a number of problems:

- Every little change in the public Sokol APIs requires a manual update to
the language bindings. Even though I try to reduce breaking source-code-level
changes to an acceptable level, changes that break binary compatibility but
not source-code-compatibility are fairly frequent (e.g. a recompile may be
required but no source code changes). Binding maintainers might decide to
accumulate many small changes until updating the bindings making the bindings
lag behind the 'official' headers, or they might loose interest in
maintaining the bindings, or small 'fringe languages' might no be deemed
"worth the effort" if too much work needs to go into maintaining the
bindings.

- In the opposite direction, having many manually maintained bindings out in the
wild might increase the 'resistance to change' in the original APIs. It is already
quite hard to decide whether a small cleanup in the public API is worth it
if it breaks existing code.

- Finally, it's much easier to introduce bugs in manually created bindings,
a simple typo can result in mysterious memory corruption errors, so in turn
a very exhaustive test suite would be needed to ensure that the bindings
are actually correct.

## ...what about automatically generated bindings?

So manually maintained bindings are "a bit" suboptimal, what options are
there for automatically generated bindings? Such a bindings generator
would require some basic information about the C-API as input:

- enums and their content
- structs and their content
- function and their signatures (arguments and return values)

With this information available, language bindings could be generated with
fairly little effort for each language by writing a script which turns
this abstract API description into a specific language binding module.

So far I had been looking into the following options, none of them are
all that great because they either make development work on the headers
more difficult, or are big software projecs on their own:

- Describing the "ground truth" API in a separate Interface Definition
Language like WebIDL, and either maintain that in parallel to the C-API (the
hassle!) or code-generate the C-API from the IDL as if it were just another
language binding (also bad since it means the final single-file C headers
would need to be code-generated for distribution, which in turn makes
development and contributions harder).

- Annotating the C-API with keywords understood by a simple parser but ignored
by the C compiler. With such annotations, a simple text-parsing tool can be
written to extract the public API information which doesn't need to be a 
full blown C parser. The downside of this approach is that it makes the 
original source file messy to read with all those annotations.

- Hacking an existing C compiler or using libllvm to extract the required
type information. This falls into the category 'big software
project on its own', definitely more work than I want to invest into a
solution.

- Using an existing language which can directly import C headers and which
provides type reflection. This is a very interesting approach which I'm
planning to revisit later. Languages like Zig (and I think Nim too) can
import C headers and provide compile-time type reflection which allows to
walk over the type information that has been imported from the C header. My
experiments in Zig only failed because of some obscure open TODO error
messages in the current Zig compiler. But generally this idea should work and
might be a better long-term solution then the method I used in the end.

All in all the best solution is to take the original C API declarations as
'ground truth' (instead of a separately maintained API description) and
'somehow' extract the type information needed to create language bindings.

## 'clang -ast-dump=json' to the rescue

I stumbled over this clang feature more or less by accident a couple weeks ago,
and wanted to see what it looks like dumping the AST of the sokol_gfx.h API,
not expecting much (because I thought that dumping the AST is more useful
for debugging clang's code generation than anything else, I always had associated
Abstract Syntax Trees more with implementation code, less with API declarations).

(Disclaimer: the output for of clang -ast-dump' isn't guaranteed to be stable,
but for the very small subset required for creating API bindings I think
this is an acceptable risk)

The result looked promising. The AST dump is a bit spammy, but all the
required information is there. E.g. this is what the ```sg_buffer``` struct
looks like in C:

```c
typedef struct sg_buffer {
    uint32_t id;
} sg_buffer;
```

The AST dump (generated with ```> clang -Xclang -ast-dump=json sokol_gfx.h```)
looks like this (just the struct declaration and ignoring the typedef):

```json
    {
      "id": "0x7f82b882a668",
      "kind": "RecordDecl",
      "loc": {
        "offset": 24628,
        "file": "/Users/floh/projects/sokol/sokol_gfx.h",
        "line": 607,
        "col": 16,
        "tokLen": 9
      },
      "range": {
        "begin": {
          "offset": 24621,
          "col": 9,
          "tokLen": 6
        },
        "end": {
          "offset": 24655,
          "col": 43,
          "tokLen": 1
        }
      },
      "name": "sg_buffer",
      "tagUsed": "struct",
      "completeDefinition": true,
      "inner": [
        {
          "id": "0x7f82b882a720",
          "kind": "FieldDecl",
          "loc": {
            "offset": 24651,
            "col": 39,
            "tokLen": 2
          },
          "range": {
            "begin": {
              "offset": 24642,
              "col": 30,
              "tokLen": 8
            },
            "end": {
              "offset": 24651,
              "col": 39,
              "tokLen": 2
            }
          },
          "name": "id",
          "type": {
            "desugaredQualType": "unsigned int",
            "qualType": "uint32_t",
            "typeAliasDeclId": "0x7f82b7883cb8"
          }
        }
      ]
    },
```

That's a lot of data I'm not really interested in, so I wrote a small
python script which compresses the clang AST-dump into just the information
needed for generating language bindings:

```json
    {
      "kind": "struct",
      "name": "sg_buffer",
      "fields": [
        {
          "name": "id",
          "type": "uint32_t"
        }
      ]
    },
```

...and the next step is to convert this simplified generic API description into
a language-specific version, for instance a Zig struct:

```zig
pub const Buffer = extern struct {
    id: u32 = 0,
};
```

...and that's basically it. The idea is to have an automated workflow
implemented in a few simple python scripts which runs on every git push (or
when creating a version tag) and which spits out a number of language bindings
which are guaranteed to match the latest changes to the Sokol APIs.

The whole idea already works quite well, e.g. this is what the Zig bindings
for Zig look like (at the time this blog post was written):

[github.com/floooh/sokol-zig/src/sokol/gfx.zig](https://github.com/floooh/sokol-zig/blob/72003b89f91e034450336efd3f9c8cc944c6a40c/src/sokol/gfx.zig)

But in the long run I want to do some subtle changes to the Sokol header APIs
to make them more 'binding friendly'. In my opinion those changes also make
the C-APIs more correct without giving up convenience.

## Binding-Friendly C-APIs

I stumbled over a few minor things when exposing the Sokol APIs to Zig (and
to some extent also earlier with C++). Some of those can be considered basic
'design warts' which should be fixed anyway (things like type casts being
required in C++ but not C), others will provide hints to the binding
generators to allow generating more 'idiomatic bindings' for higher-level
languages.

Currently this is just a loose collection of ideas, in the future these probably
become a bit more formal "API design guidelines" for the Sokol headers:

- top-level constants should be defined in an anonymous enum, not in preprocessor defines
- nested anonymous structs and unions must be avoided
- all indices and sizes should be unsigned (and probably size_t)
- pointer/size pairs should be combined in a 'range struct', this may allow language
bindings to replace those with language-specific range- or slice-types
- nullable and non-nullable pointers should be wrapped in separate typedefs as
a hint for the bindings-generator
- ...generally use typedefs to communicate 'intent' to the bindings generator
- functions where it makes sense to be called with integer or float parameters 
(and which rely on C's implicit type conversion) should get separate versions
(for example sg_apply_viewport(), which takes integer parameters, but which is
also often called with float parameters)

...and so on. The basic idea is that exposing the Sokol C-APIs to as many 
languages as possible may also help feeding improvements back into the 
C-APIs, basically some sort of cross-language symbiosis. The tricky part
here is to make the API more correct but without making it less convenient 
to use from C.

## Why generate separate bindings for Zig?

One of Zig's greatest features is that it can directly import C headers, so
why go through all the hassle and do a separate semi-automatic language
generation via clang's ast-dump?

The answer is that a semi-automatic approach with a per-target-language
python script inbetween can contain special-case code to generate more
'idiomatic' bindings which feel more like a native language module and less
like an imported C module.

So far I only scratched the surface and just tried to rename the API elements
to follow Zig's naming conventions. This simple feature already does a lot to
not make the code feel 'alien', for instance this C frame callback function
of a simple draw pass via sokol_gfx.h:

```c
void frame(void) {
    sg_begin_default_pass(&(sg_pass_action){0}, sapp_width(), sapp_height());
    sg_apply_pipeline(state.pip);
    sg_apply_bindings(&state.bind);
    sg_draw(0, 3, 1);
    sg_end_pass();
    sg_commit();
}
```

...looks like this in Zig:

```zig
export fn frame() void {
    sg.beginDefaultPass(.{}, sapp.width(), sapp.height());
    sg.applyPipeline(state.pip);
    sg.applyBindings(state.bind);
    sg.draw(0, 3, 1);
    sg.endPass();
    sg.commit();
}
```

...and here is an example of what I want to achieve by adding special typedef-hints
to the C-API. Creating a vertex buffer from a float-array in C looks like this:

```c
    const sg_buffer vbuf = sg_make_buffer(&(sg_buffer_desc){
        .content = vertices,
        .size = sizeof(vertices),
    });
```

...currently the same looks like this in Zig:

```zig
    const vbuf = sg.makeBuffer(.{
        .content = &vertices,
        .size = @sizeOf(@TypeOf(vertices))
    });
```

...some things are more convenient than the C version, but others are more 
awkward and wouldn't be necessary in a proper Zig API (like the & and the
@sizeOf(@TypeOf(...))).

Ultimately I want it to look like this (replacing the "raw" pointer/size pair
with a Zig slice type). This will require changing the C API to have a combined
pointer/size-pair type (or a lot of special-case hacks in the binding-generator
scripts, this would be an option too, but one I'd like to avoid):

```zig
    const vbuf = sg.makeBuffer(.{
        .content = vertices
    });
```

...and for other languages I hope to achieve a similar 'idiomatic' style
through the per-language binding-generator scripts. For instance I'm also
thinking about special C++ bindings even though the Sokol headers are usable
from C++ just fine right now. But they are not as convenient to use from C++
as from C99, because it turned out that the limited designated initialization
in C++20 isn't all that useful for initializing non-trivial data structured
with dozens of members and nested structs and arrays. So it probably makes
sense to create C++ bindings which implement the builder pattern instead of
relying on designated initialization. Another option might be to convert the
raw C callbacks with a userdata pointer into C++-lambda friendly
std::function objects.

But these are all just ideas at the moment. The good thing is that it's fairly
easy to try out those ideas, since they can be realized with a small python
script which reads the simplified API description JSON file and spits out
a generated source file, it's very easy to iterate with such a setup.

Currently I'm not in a hurry to merge the code-generation experiments into
the master branch (they are here in the [bindgen branch](https://github.com/floooh/sokol/tree/bindgen/bindgen)).
I'll probably create at least one or two other binding-generator scripts for other
languages first, and then do a cleanup round on the Sokol APIs (which means
breaking changes). The long-term goal is to run the binding generation fully
automated via CI (like github actions).

And that is all for today :)
