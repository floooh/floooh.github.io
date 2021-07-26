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

Like many modern languages, Zig has builtin testing capabilities which is very
easy to use. One can just write tests in regular implementation files, and run
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
Those initial tests written alongside the implementation are great for catching
some obvious problems without delaying too far into the future, but they 
are by far not as thorough as specialized instruction testers.

The further I got with the project, the less I relied on such small tests, and
instead I moved to higher level standard test programs written on Z80 hardware
like ZEXALL/ZEXDOC (implemented here: https://github.com/floooh/kc85.zig/blob/main/tests/z80zex.zig).

## The Zig Project Structure



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