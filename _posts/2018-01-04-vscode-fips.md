---
layout: post
title: Better VSCode support in fips
---

I just merged support for latest [Visual Studio Code](https://code.visualstudio.com/) features back into
the [fips](https://github.com/floooh/fips) master branch which
make for a quite pleasent (and consistent!) C/C++ development experience
on OSX, Linux and Windows.

### How to use fips with VSCode

Apart from the minimal fips requirements (cmake and python) you
need Visual Studio Code (obviously) and on OSX and Linux the
ninja build tool in the path:

```
> code --version
1.19.1
> ninja --version
1.8.2
```
You also need the official [C/C++](https://code.visualstudio.com/docs/languages/cpp) and 
[Python](https://code.visualstudio.com/docs/languages/python) extensions.

Also make sure to update fips to the latest version (run **git pull** in the
fips directory)

To use VSCode as IDE, cd into a fips project and activate one of the
```*-vscode-*``` build configs:

```
# on OSX:
> ./fips set config osx-vscode-debug
# on Linux:
> ./fips set config linux-vscode-debug
# on Windows:
> fips set config win64-vscode-debug
```

Next run ```./fips gen``` and ```./fips open``` as usual:

```
> ./fips gen
...
> ./fips open
>
```

Visual Studio Code should start with the fips project loaded.

### Multi-Root Workspace Support

Fips now supports VSCode [Multi-root Workspaces](https://code.visualstudio.com/docs/editor/multi-root-workspaces) to include all
dependencies into the VSCode project explorer:

![fips-vscode-1](../../../images/fips-vscode-1.png)

Most importantly, fuzzy file search works for dependencies:

![fips-vscode-2](../../../images/fips-vscode-2.png)

...as well as fuzzy symbol search:

![fips-vscode-3](../../../images/fips-vscode-3.png)

### Building

Fips creates a VSCode build task for each cmake target, everything
works as expected from there on:

- press Ctrl/Cmd+Shift+B to build the entire project (cmake's ALL target)
- press Ctrl/Cmd+P and start typing "task " (with space at the end) to fuzzy-search for build targets and build one of them
- or use the new ```Tasks -> Run Task...``` menu item to select a single build task

### Debugging

Fips creates one debug target per executable build-target, and several debug
targets for python scripts used in fips projects:

![fips-vscode-4](../../../images/fips-vscode-4.png)

Unsurprisingly, selecting one of the C/C++ build targets starts the C/C++
debugger (which uses lldb, gdb or the Visual Studio debugger under the hood):

![fips-vscode-5](../../../images/fips-vscode-5.png)

Debug targets that start with **fips** are python scripts, **fips codegen**
is the fips code-generation custom build job, and the others are custom fips
commands (for instance the sokol-samples project has a custom command 'webpage'
which is used to build and test the asm.js/wasm samples webpages).

Some fips custom commands need additional parameters, these must be added
manually to the launch.json file:

![fips-vscode-6](../../../images/fips-vscode-6.png)

From there on, python debugging works as usual:

![fips-vscode-7](../../../images/fips-vscode-7.png)

### Header Search Paths

Header search path resolution for system headers 
on Windows and Linux is currently not perfect. So far I only
found a good solution to list the system header search paths
for clang on OSX ([see here](https://github.com/floooh/fips/blob/30bf246674dcf7853489dfdef6a7bc91bf75d1cd/mod/tools/vscode.py#L85)). If you know of a similar solution for Visual Studio
on Windows and GCC on Linux, let me know! PRs are also welcome of course :)

And that's all for today, happy hacking!
