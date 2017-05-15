---
layout: post
title: "Oryol: the new SPIRV-based shader pipeline"
---

Next weekend I will merge the new SPIRV-based shader code generation back
into the master branch. The work is happening in the 'spirv-tooling' branch
and is tracked in [Pull Request
\# 253](https://github.com/floooh/oryol/pull/253).

Most of this was fairly un-exciting build-system-plumbing (just like most engine programming actually), but I think it's laying a nice and solid foundation for future
shader pipeline work (not sure yet what will come next though).

All the interesting stuff is actually happening in two Khronos tools/libs, I'm more-or-less just glueing those together:

- [glslang](https://github.com/KhronosGroup/glslang): the contained standalone tool glslangValidator is used to compile GLSL into SPIR-V bytecode (I already used this tool before in Oryol, but only to check shader code for errors).
- [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross): this is used as library in an Oryol-specific wrapper tool (called oryol-shdc) to translate the SPIR-V bytecode to various shader languages and to write out reflection information

There are a few minor issues remaining in SPIRV-Cross, but they can be
worked-around and no code changes in Oryol will be required when those
fixes land (more details at the end of this file).

The merge will break existing shader code and the C++ code that deals with uniform updates. Details 
on how to fix those problems can also be found at the end.

The pay off for those breaking changes is that shader code is now written in
standard-GLSL (v330) instead of some cobbled-together GLSL/HLSL-bastard dialect
with brittle text-macro-magic. Also, uniform updates on WebGL and GLES2 are
now much more efficient for non-trivial shaders \o/

### Oryol Shader Coding Overview

This is a quick recap how shader programming in Oryol looks and feels. 
From the outside this hasn't actually changed that much.

Shaders in Oryol are program code, not asset files. This means that 
shader files live next to C/C++ sources in a project, and are 'compiled' and 'linked' into executables.

Shader source files are (now) written in standard GLSL, interspersed with
(much fewer than before) '@-tags' which identify vertex- and fragment-shaders,
and define how those are linked into shader-programs. A very simple shader
file might look like this (in the new syntax):

```text
@vs myVS
uniform params {
    mat4 mvp;
};

in vec4 position;
in vec4 color0;
out vec4 color;

void main() {
    gl_Position = mvp * position;
    color = color0;
}
@end

@fs myFS
in vec4 color;
out vec4 fragColor;
void main() {
    fragColor = color;
}
@end

@program MyShader myVS myFS
```

This defines one vertex- and one fragment-shader, and links them
together into a shader program called 'MyShader'.

This file is added to the build process with a CMakeLists.txt macro ```oryol_shader()```, for instance:

```text
fips_begin_app(Shapes windowed)
    ...
    oryol_shader(shaders.glsl)
    ...
fips_end_app()
```

The cmake macro ```oryol_shader()``` is a wrapper for the generic code-generation
function ```fips_generate()``` ([for more info on fips see here](http://floooh.github.io/fips/)).

The shader source files are added to IDE projects (e.g. in Xcode or
Visual Studio), along with the generated C++ header/source pair:

![oryol-shd-xcode](../../../images/oryol-shd-xcode.png)

Shader source code is validated during compilation, and
errors show up in the IDE like normal C++ compiler errors:

![oryol-shd-error](../../../images/oryol-shd-error.png)

On the C++ side, shader uniform blocks are turned into C structures,
with the same struct and member names as on the GLSL side:

```cpp
    MyShader::params ub;
    ub.mvp = computeModelViewProj();
    Gfx::ApplyUniformBlock(ub);
```

### How the new Shader Code Generation works

The Shader.py code-generator script is the 'big orchestrator', it is invoked as custom
build job by the build process, reads the shader source file, invokes a number of command line tools, parses their error messages, does a number of
additional validations on their output and finally generates a C++ source/header pair which is then compiled as usual:

<img src="https://docs.google.com/drawings/d/19CG5ZuOQ9JSwF-RE0Od0CgVCTmsmyQlEkEMCJ4fO-Zo/pub?w=517&amp;h=720">

(tools are red, intermediate files blue, output files yellow)

At the top in green is the hand-written shader source file in GLSL v330
format, this is first split into individual vertex- and fragment-shader 
files (also in GLSL v330).

These intermediate GLSL files are then compiled to SPIR-V bytecode via
the [glslangValidator tool](https://github.com/KhronosGroup/glslang),
any syntax errors are caught here, mapped back to the 
original input file and printed in a format matching C++ compiler error messages (so that IDEs can understand them).

The resulting SPIR-V bytecode file is then fed into [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) (or rather my own [command
line wrapper tool](https://github.com/floooh/oryol-tools/blob/master/src/oryol-shdc/main.cc) around SPIRV-Cross as library).

The SPIRV-Cross step generates output files in the following shader
language dialects (depending on the current build config):

* GLSL 100 (for GLES2 and WebGL)
* GLSL 300 ES3 (for GLES3 and WebGL2)
* GLSL 330 (for desktop GL)
* HLSL5 (for D3D11)
* MetalSL (for Metal on OSX and iOS)

If the build config uses Metal or D3D11 as rendering backend, the generated
source files will be compiled right away into Metal or HLSL byte code by
invoking the respective shader compilers, and the result will be written into a
C-header as a hexdump.

In addition to the generated dialect shader files, the SPIRV-Cross wrapper tool also writes a JSON file with reflection information needed for the C++ side with detailed info about the vertex shader inputs, uniform blocks and textures
used by the shader, this information is for instance used to generate C-structures matching the shader uniform blocks:

```json
{
    "stage": "vs",
    "uniform_blocks": [{
        "type": "params",
        "name": "_19",
        "slot": 0,
        "size": 64,
        "members": [{
            "name": "mvp",
            "type": "mat4",
            "num": 1,
            "offset": 0,
            "matrix_stride": 16
        }]
    }],
    "textures": [],
    "inputs": [{
        "name": "position",
        "type": "vec4",
        "slot": 0
    }, {
        "name": "color0",
        "type": "vec4",
        "slot": 10
    }],
    "outputs": [{
        "name": "color",
        "type": "vec4"
    }]
}
```

Finally the Shader.py python script takes over again and performs the final
steps to generate the C++ source and header file which will then be compiled
into the executable.

### Faster Uniform Updates for WebGL / GLES2

A nice side effect of the new shader pipeline is that uniform updates in
GLES2 and WebGL are now much more efficient for uniform blocks with multiple
members. Each GLSL uniform block is now converted to a single vec4-array in
the output code (only for GLSL outputs), which can be updated with a single
```glUniform4fv()``` call instead of one ```glUniformXX()``` call per
uniform block member. This is a good thing for WebGL which has a higher
call-overhead than native GL implementations.

### Limitations and Known Issues

These are the current limitations and known issues:

- **Cannot use integers or booleans in uniform blocks**: This is because uniform blocks are now converted to vec4-arrays in GLSL (the actual limitation in SPIRV-Cross is that integer and float types cannot be mixed in the same uniform block, but for simplicity's sake I'm disallowing integers and booleans in uniform blocks alltogether, at least for now).
- **Arrays in uniform blocks are limited to mat4, mat2 and vec4**: The reason is that all items in uniform block arrays are padded to 16 bytes anyway, so smaller types have a lot of waste. 
- **mod() is currently only implemented for scalars in HLSL**: this is currently a bug in the SPIRV-Cross HLSL backend, tracked here: [SPIRV-Cross #173](https://github.com/KhronosGroup/SPIRV-Cross/issues/173)


Finally there's currently a hack required in my own fork of SPIRV-Cross in
the HLSL backend to correctly handle RowMajor vs ColMajor matrices, this
is tracked here: [SPIRV-Cross #170](https://github.com/KhronosGroup/SPIRV-Cross/issues/170). Once this is
resolved I will switch back from my own fork to the official upstream version.

### Required Code Changes

In general all required code changes should be caught by compile errors, so finding and fixing them should be fairly straight-forward.

These are the required changes on the shader-side:

- the ```@code_block``` tag to define a section of code for inclusion in a vertex- or fragment-shader has been simplified to ```@block```
- it is no longer allowed to nest ```@block```'s inside each other
- ```@use_code_block``` has been renamed to ```@include``` and included code can now be placed anywhere inside a ```@vs``` or ```@fs``` block
- the following @-tags have been removed: ```@uniform_block``` ```@texture_block``` ```@use_uniform_block``` ```@use_texture_block``` ```@in``` ```@out``` ```@highp```
- the following compatibility macros have been removed and replaced with:
    - ```_vertexid``` => gl_VertexID
    - ```_instanceid``` => gl_InstanceID
    - ```_position``` => gl_Position
    - ```_pointsize``` => gl_PointSize (ignored on D3D11)
    - ```_color```, ```_color1```, ```_color3```, ```_color4```: use 'modern GLSL' fragment shader outputs instead (e.g. ```out vec4 color;```, or for MRT rendering with explicit location ```layout(location=1) out vec4 c1;```)
    - ```_fragcoord``` => gl_FragCoord
    - ```_const``` => const
    - ```_func``` => nothing
    - ```mul()``` => nothing, just use ```*```
    - ```tex2D(), ...``` => use the standard GLSL texture() built-ins

On the C++ side, the generated uniform block C-structs and the contained members don't have a separate name, instead they are named exactly like their shader counterparts. To avoid naming collisions, the structs live inside a namespace named after ```@program``` tag in the shader source. 

For instance:

```text
@vs vs
uniform bla {
    mat4 mvp;
    vec4 color;
};
...

@program MyShader vs fs
```

...will generate a C-struct like this:

```cpp
namespace MyShader {
    struct bla {
        glm::mat4 mvp;
        glm::vec4 color;
    };
}
```

And that is pretty much all :) 