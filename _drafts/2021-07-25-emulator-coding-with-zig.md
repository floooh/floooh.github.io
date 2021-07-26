---
layout: post
title: Emulator Coding with Zig
---

I wrote a KC85 home computer emulator in Zig during the last 3 weeks to get a bit more
familiar with Zig for mid-sized projects. This is my 4th or 5th iteration
of a KC emulator, which makes it a useful "toy project" for trying out
new languages, because I have the "inner workings" already figured out and
can focus on how the emulation maps to language features.

The github project is here:

https://github.com/floooh/kc85.zig

## The KC85 home computer series

I'll keep this and the next section short, because the focus of this blog post
should be on Zig. But it makes sense to know a few basics about the emulated
system and how the emulator works.

The KC85 (more specifically, the KC85/2, /3 and /4) was a series of East German
8-bit computers built between 1984 and 1989. The hardware was built around the
a Z80-compatible CPU clocked at 1.75 MHz, one Z80 PIO, one U857 (== Z80 CTC).
and 32+8 KBytes to 128+20 KBytes of RAM and ROM. The video hardware was completely
hardwired and produced a PAL image with 320x256 pixels.

Unlike most Western home computers, East German computers had no custom chips,
everything was built from standard parts. This reflected the priorities in the
East German semiconductor industry, the focus was on office computers and
microcontrollers for other industries, education or even home computing
was merely a byproduct. This is (most likely) also the reason why the KC series
was clocked so low, the chips used in those computers were "waste chips"
that couldn't run at full speed.

Hardware-wise the KC series was clearly inspired by the ZX Spectrum, but 
without being a Speccy clone. Some aspects were worse (like the CPU speed)
and some were better (like the pixel- and color-resolution).

## How the emulator works

Nothing new here compared to my earlier iterations, only the Z80 instruction
decoder is "hand crafted" and doesn't rely on code generation like in 
my ['chips' emulators](https://github.com/floooh/chips) (code generation is an 
area I want to explore later with Zig).

Unlike my newer 6502 emulators, the Z80 based emulator is still
"instruction stepped", not ["cycle stepped"](https://floooh.github.io/2019/12/13/cycle-stepped-6502.html)
 (which means the CPU emulation can't be suspended in the middle of an instruction).

From within the CPU instruction decoder loop, a 'system tick callback' is
called at leat once per instruction (specifically: once per memory or
IO machine cycle).

The system tick callback is responsible for performing memory- and IO-
read- and write-accesses, and generally ticking the rest of the system
forward (decode video memory and tick the other chips in the system).

Communication between the various system components is handled through a 
64-bit 'pin bit-mask', just like real hardware chips communicate with
the outside world through the input/output pins, the chips emulators
communicate through a bit mask in a single 64-bit integer, with one bit
for each pin. Wiring the chips together is then just a matter of clearing
or setting the right pins before calling each chip's tick function, and
then inspecting the returned pin mask.

This pin-bit-mask based emulation has one pretty big advantage: once the
chips emulators are in place it is fairly trivial to build an emulated
system from the original motherboard schematics. One basically just 
needs to follow the wires on the schematics from chip to chip and
map this wiring to pin-bits in the emulation (it's a bit simplified,
but all in all it's surpring how well this works for radically
different systems, this is definitely an idea that worked out very well).

## Testing with Zig

Like many modern languages, Zig has a built-in testing feature which is very
easy to use. One can just write tests in regular source files, and run
them with "zig test src.zig". This is very handy when building a project
bottom-up when there's just a handful of unrelated source files that don't yet
work together. For instance most of the basic building blocks in the CPU
emulation consisted of writing a small functions like this:

```zig
fn add8(r: *Regs, val: u8) void {
    const acc: u64 = r[A];
    const res: u64 = acc + val;
    r[F] = addFlags(acc, val, res);
    r[A] = @truncate(u8, res);
}
```

...and then immediately write a test like this:

```zig
test "add8" {
    var r = makeRegs();
    r[A] = 0xF;
    impl.add8(&r, r[A]); try expect(testAF(&r, 0x1E, HF));
    impl.add8(&r, 0xE0); try expect(testAF(&r, 0xFE, SF));
    r[A] = 0x81; 
    impl.add8(&r, 0x80); try expect(testAF(&r, 0x01, VF|CF));
    impl.add8(&r, 0xFF); try expect(testAF(&r, 0x00, ZF|HF|CF));
    impl.add8(&r, 0x40); try expect(testAF(&r, 0x40, 0));
    impl.add8(&r, 0x80); try expect(testAF(&r, 0xC0, SF));
    impl.add8(&r, 0x33); try expect(testAF(&r, 0xF3, SF));
    impl.add8(&r, 0x44); try expect(testAF(&r, 0x37, CF));
}
```
Those simple initial tests written alongside the implementation are great for catching
some obvious problems for such a "brain in a jar" CPU but it's no replacement
for much more thorough instructions testers which have been written and tested
on real hardware. 

The further I got with the project, the less I relied on such small tests, and
instead I moved to "established" standard test programs like
[ZEXALL/ZEXDOC](https://github.com/floooh/kc85.zig/blob/main/tests/z80zex.zig).
This source file implements a minimalistic CP/M environment, just enough to run
the ZEX tests and dump its output to the terminal (CP/M was the standard
operating system of the ancient times even before MS-DOS was a thing).

## The Zig Project Structure

I started out with a flat list of source files, one file per module, one
module per emulator component, and one module for the actual system emulator
at the top of the dependency tree. The module dependencies look like this:

```
- kc85.zig -+
            +-- z80.zig
            +-- z80ctc.zig -+
            +-- z80pio.zig -+
            |               +-- z80daisy.zig
            +-- memory.zig
            +-- beeper.zig
            +-- clock.zig
            +-- keybuf.zig
```

This is the entire emulator as "pure" platform-agnostic Zig code.

- ```z80.zig```, ```z80ctc.zig``` and ```z80pio.zig``` are the three chip
  emulators, with the CTC and PIO sharing the same interrupt code in
  ```z80daisy.zig``` (for "daisy-chain" which was the common name for the
  fairly sophisticated Z80-family-chip interrupt handshake protocol which
  automatically managed a priority "daisy chain" of interrupt sources without
  intervention from the CPU.

- ```memory.zig``` implements a simple layered page table which maps a 16-bit
  address space to host memory with a 1 KByte page size. This seems a bit
  overkill for a simple 8-bit home computer, but the KC85 has fairly
  sophisticated expansion module system which (theoreticall at least) allowed
  to map 4 MBytes(!) of RAM or ROM into the 64 KByte address space of the Z80
  via bank switching.

- ```beeper.zig``` is the simple general square-wave oscillator which produces
  a mono sample stream to be pushed into a separate audio backend. The KC85 had
  stereo sound, so it needs two of those beepers.

- for ```clock.zig``` and ```keybuf.zig``` it's a bit debatable where they
  belong, both could just as well live on the host-bindings side (which I
  haven't mentioned yet).  ```clock.zig``` converts real-world micro-seconds
  into system-specific clock ticks, and keeps track of "remainder ticks"
  because the CPU emulation can only run full instructions, and
  ```keybuf.zig``` is a little helper class to keep short key presses around
  for at least one host system frame so that the emulator is guaranteed to
  notice them.

After most of the emulator was in place, I decided to split the source directory
into subdirectories (and thus "Zig packages"), to separate the platform-agnostic
emulator code from the "host system bindings". Those host bindings take care
of:

- rendering the emulator's video output into a window
- making the emulator's audio output audible
- feeding the emulator with keyboard input
- keeping track of real-world time in order to run the emulator at the correct speed
- and finally secondary things like parsing command line arguments and loading
ROM and tape image files from the host's file system

[to be continued]


TODO:

- Zig project structure (where to put things, modules and packages)
- How modules are structured internally (what to put into structs and namespaces)
- Why the "C-header-like" module structure
- Initialization questions: heap or static objects and Zig allocators, if- and
switch-expressions to conditionally initialize data "right in place"
- Arbitrary width integers for hardware registers, carry bits, exhaustive switches etc
- Host bindings, Zig standard lib, sokol headers, errors vs optionals
- namespaced constants vs enums vs (hypothetical 'bit-twiddling bitfields')
- nice to have: proper freestanding functions?
- Zig tests versus test programs
- ???