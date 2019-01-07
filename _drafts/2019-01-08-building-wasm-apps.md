---
layout: post
title: How I build small WebAssembly apps
---

A top-to-bottom overview of what my development-workflow looks like
for building small WebAssembly applications like the [Tiny
Emulators](https://floooh.github.io/tiny8bit/).

As always, don't take it as gospel, it works well for me on my hobby
projects. It doesn't mean that the same ideas would scale to a large
professional team (although it works quite well for very small teams of
somewhat experienced programmers and artists).

On the other hand: big teams can't efficiently build small 
things anyway. Big teams tend to produce a lot of code and data.

## Languages and Tools

Currently my default-language of choice is C, sprinkled with other
languages as needed (C++, Objective-C, Javascript and Python):

- **C++**: when working with libraries that have a C++ API (in the 
Tiny Emulators, that's only for the optional debugging UI implemented
with Dear ImGui)
- **Objective-C**: this is needed in the platform-wrapper parts when
compiling the native macOS or iOS versions
- **Javascript**: similar usage as Objective-C on Apple platforms, but 
for talking to Web APIs.
- **Python**: this is only used in the build system for code generation
and build automation

In the near future, I expect that I will stick with C for library-style code,
but experiment with various small "better-C" languages for the
application-level code.

I prefer to work on Mac (default) or Linux for WASM stuff, because working
with the emscripten SDK on Windows can be 'a bit unpleasant' (for some reason
clang on Windows is much slower than on Linux or Mac, and weird Windows
'features' like the 8192 character command line length limit).

For actual programming work I prefer to work in a IDE because of the integrated
debugger. On Mac this is mostly VSCode, but at least a few times per day 
I fall back to Xcode to run the clang analyzer, address- and undefined-behaviour-
sanitizers, or the Metal graphics debugger.

## The Edit-Compile-Test Loop

This looks a lot like working with Rust's cargo. I'm dogfooding my own
cmake 'workflow enhancer' called [fips](http://floooh.github.io/fips/)
(yeah I really need to update the documentation front page).

Everything starts in the command line, but most of the actual work happens
in VSCode or Xcode (or Visual Studio when I'm on Windows).

The integrated terminal in VSCode is **extremely** useful.

Even though my focus is on producing a WebAssembly application, 90% of the 
time I'm developing, building and debugging a traditional native macOS/Linux/Windows 
application. The main reason for this is fast compiles (typically under a second),
and seamless debugging (which is **very** important for me, typically 
I write maybe a pageful of code, set a few breakpoint, and step through the
code I just wrote).

Maybe once an hour, or when I have finished a small feature I'm testing the
WebAssembly build. Which brings me to **fips**.

**fips** is essentially a python script which provides a cargo- or 
npm-like workflow on top of cmake:

- manage different build configurations for cmake
- simplify working with external git dependencies
- python scripting for project-specific code generation and build commands

For instance:

If I want to start working in Xcode, I run:

```
> ./fips set config osx-xcode-debug
> ./fips open
```

This selects a 'build config' (which is just a [small text
file](https://github.com/floooh/chips-test/blob/master/fips-files/configs/osx-xcode-debug.yml)
with a bunch of cmake options) to build for OSX, generates Xcode project
files, and by default builds for debugging.

The ```./fips open``` command opens Xcode into the current project and 
build config (and invokes cmake if this hasn't happened yet).

But usually I prefer VSCode, so let's use this instead:

```
> ./fips set config osx-vscode-debug
> ./fips open
```

As you can see, the difference between using VSCode instead of Xcode is just a different build config name.

If I just want to build the project on the command line instead, I run:

```
> ./fips build
```

If I was still on the ```osx-xcode-debug``` build config, this builds via
xcodebuild, which has a spammy output. Let's use ninja instead, and build the
optimized release version:

```
> ./fips set config osx-ninja-release
> ./fips build
```

After the build is successful, I can simply run a compiled executable by its target name, e.g.:

```
> ./fips run cpc
```

If I don't know what build targets are in the project, I can get a list of targets:

```
> ./fips list targets
```

Now the more interesting stuff! If I want to check the WASM build, I do 
the same command line build as above, but for a different build config:

```
> ./fips set config wasm-ninja-release
> ./fips build
> ./fips run cpc
```

This compiles the project via the emscripten SDK, and 'runs' the build
target ```cpc``` by starting a local web server and web browser.

(I ommitted installing the emscripten SDK, which happens with ```> ./fips setup emscripten```).

...so that's my normal development loop, most of the development work
(and debugging!) happens on the native OSX version (or Linux, or Windows)
in an IDE, and from time to time I cross-compile and test the WebAssembly 
version.

When I'm done for the day, I usually update the Tiny Emulators webpage on github-pages. For
this I have written a [small (and messy) build-automation python
script](https://github.com/floooh/chips-test/blob/master/fips-files/verbs/webpage.py)
which build the project for WebAssembly and builds a static website from the
result:

```
> ./fips webpage build
```

And to do run a local test of the webpage:

```
> ./fips webpage serve
```

Just like running a local WASM program, this will start a local web server
and a browser pointing to the local Tiny Emulators main page.

And that's it for my 'outer' development loop. I think it's quite pleasent 
for a C/C++ project (not quite as pleasent as hot-code-reload, but since
compile-times are hardly noticeable that's not much of an issue). 

I just wish the debugging would start a lot faster both in VSCode and Xcode,
this has regressed dramatically since Xcode10 (or late Xcode9?), it takes nearly
10 seconds for the first start, and 1 or 2 seconds for followup starts. This
used to be pretty much instant.

