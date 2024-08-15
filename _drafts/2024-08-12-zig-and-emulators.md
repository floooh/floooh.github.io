---
layout: post
title: "Zig and Emulators"
---

> NOTE: I made several attempts for this blog post and it always ended up
> getting too big and rambly. As a result I'm omitting all background info
> about how the emulators actually work and will focus on Zig language
> features instead. A lot of this stuff either already exists in older blog posts, or
> should exists in other blog posts not yet written :)

Some quick Zig feedback in the context of a new 8-bit emulator project I started
a little while ago:

[https://github.com/floooh/chipz](https://github.com/floooh/chipz])

Currently this includes:

- a cycle-stepped Z80 CPU emulator (similar to the emulator described
  here: https://floooh.github.io/2021/12/17/cycle-stepped-z80.html)
- chip emulators for Z80 PIO, Z80 CTC and an AY-3-8910 sound chip
- system emulators for Bombjack, Pengo and Pacman arcade machines,
  and the East German KC85/2../4 home computer series
- a code generation tool to create the Z80 instruction decoder code block
- various tests to check Z80 emulation correctness

With the exception of an external C dependency for 'host system glue'
(the cross-platform sokol headers used for the window, input, rendering
and audio output), the project is pure Zig (unlike the original project
which is a mix of C, C++, Python for code generation and cmake scripts
for the build system).

Eventually I hope to bring the project on paar with, but this will take
a while (and next I'll go back to sokol-gfx feature development):

[https://github.com/floooh/chips](https://github.com/floooh/chips)


## Dev Environment

I'm coding on an M1 Mac in VSCode with the [Zig Language Extension](https://marketplace.visualstudio.com/items?itemName=ziglang.vscode-zig), and [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)
for step-debugging.

The Zig and ZLS (Zig Language Server) installation is managed with ZVM (https://github.com/tristanisham/zvm).

For the most part this setup works pretty well, with a few tweaks:

- I have setup 'build on save' to get more complete error information as described here:
  [Improving Your Zig Language Server Experience](https://kristoff.it/blog/improving-your-zls-experience/)
  (I'm not bothering with creating separate non-install build targets though)
- With the default Zig VSCode extension settings I was seeing that in long coding
  session (5..6 hours or so) saving would take longer and longer until it would
  eventually get stuck. After asking around on the Zig Discord this could be solved
  by explicitly setting the Zig Language Server as 'VSCode Formatting Provider'
  in the Zig Extension settings.
- When debugging, there's a somewhat annoying issue that the debug line information
  seems to be off in some places, the debugger appears to step into the last
  line of an inactive if-else block for instance. Again, Discord to the rescue,
  this seems to be a known regression.

All in all, not yet perfect, but good enough to get shit done.

## Zig Comptime and Generics

## Debug Performance

## Bit Twiddling and Integer Maths

## Strings, Slices and Memory
