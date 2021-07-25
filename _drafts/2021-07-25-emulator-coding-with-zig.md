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

The address decoding for chip-selection, chip-IO and memory banking was
implemented with simple and standard logic chips on the KC85/2 and /3 and
partly with a PROM on the KC85/4 (so unlike most Western home computers, there
were *no* custom chip at all in this system).

The KC85/2 and /3 had 16 KByte of system RAM, 16 KByte hardwired video RAM and
the operating system in an 8 KByte ROM at the end of the address space.

The KC85/3 came with a builtin BASIC interpreter in ROM.

The KC85/4 was a true memory monster with 64 KByte general RAM, and 64 KByte 
video RAM (using bank switching to map 16 KByte at a time into the CPU
address space - somewhat simplified, because only 10 of the 16 KByte were actually
available to the CPU, the remaing 6 KByte were always mapped to the first 
video memory bank).

Hardware-wise, the KC series was clearly inspired by the ZX Spectrum, especially
the video decoding. Like the ZX Spectrum, the KC85 series used separate 
color attribute bytes to reduce the video hardware's bandwidth and memory
requirements. The KC had some improvements over the Speccy though:

1. The display resolution was higher, 320x256 instead of 256x192.
2. One color attribute byte covered 8x4 pixels on the KC85, versus 8x8 pixels on the Speccy.
    On the KC85/4, the color attribute resolution was increased to 8x1 pixels.
3. The color palette was more interesting and aesthetically pleasing than on
   the Speccy: 15 foreground colors, and a separate set of darker 8 background
   colors, plus one bit for blinking (which was quite useless but could be used
   for an interesting [viusal effect](https://floooh.github.io/2017/01/14/yakc-diamond-scroll.html).

On The KC85/2 and KC85/3 pixel and color addressing was cursed though. The
320x256 display was split into two areas, a left area with 256x256 pixels
(arranged at 32 byte-columns and 256 rows), and a right area with 64x256 pixels
(8 columns x 256 rows). Additionally (similar, yet different from the Speccy),
horizontal pixel lines weren't linearly arranged. Long story short, the video
memory layout made graphics rendering a nightmare, and this showed in the slow
display update routines in the operating system. For instance clearing the screen
with the OS functions took several seconds and when scrolling one could literally
watch the CPU working.

The pixel addressing issue was entirely fixed on the KC85/4 with a genius
design solution I haven't seen anywhere else so far: The video memory was
simply "rotated" by 90 degrees! Incrementally writing bytes to video memory
would fill vertical columns on screen. This totally makes sense because the
vertical display resolution is 256 (a nice round number), while the horizontal
resolution isn't a round number at 320 pixels (or 40 'byte columns', each byte
describing 8 pixels).

This 'vertical' video memory layout means one can simply put the pixel row
number (0..255) into the low 8-bit register of a 16-bit register pair, and the
column number (0..39) into the high-byte register and voila, the 16-bit
register now is the byte offset into video memory, no extra math needed. And
since video memory always starts at address 0x8000, just replace the column
number 0..39 with 128 (== 0x80) to 128+39=167 (== 0xA7), and now the 16-bit
register is the absolute video memory address between 0x8000 and 0xA7FF.

To blit an 8x8 pixel character, just point HL to the glyph data (8 bytes), and
DE to the target video memory address, and then run an unrolled loop of 8 LDI
instructions (LDI copies one byte from (HL) to (DE) and then increments both
HL and DE).

The KC85 series had no dedicated sound chip, instead two CTC counter channels
were hardwired to two piezo-beepers which generated a very rough square wave
sound. By reprogramming the CTC counters from tight CPU loops one could still
generate quite interesting sound effects, but at the cost of a high CPU
usage.

But I disgress, enough about the hardware :)

## How the emulator works

Nothing new here compared to my earlier versions, only the CPU instruction
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
- ???