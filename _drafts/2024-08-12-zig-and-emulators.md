---
layout: post
title: "Zig and 8-bit Emulators"
---



!!!KUERZEN!!!



In the last few of months I took some time off from Sokol-header feature implementation
and started a new 8-bit computer emulator project to explore what advantages
Zig has over C for writing emulators of 80's 8-bit computer systems, and to figure
out whether it makes sense to switch from C to Zig for this kind of stuff.

The TL;DR answer is a resounding 'maybe' :D There are many area where Zig is
clearly better than C, and a handful of things that are currently more awkward in Zig
than in C - however, some of those areas where Zig is more awkward than
C are examples of where C is trading correctness for convenience, so it's not
clear yet whether those things will (or even should) change in Zig.

Of course there's also my own subjectivity and my mind being tuned to the 'wrong
languages' (currently mostly C and Typescript - although I feel that this
specific combination is a pretty good starting point for Zig, superficially Zig
looks like 'C with Typescript syntax'), OTH I may not be blind yet
to awkward Zig details which seasoned Zig programmers might no longer notice
- after all, this is definitely a problem I see in my C programming: I'm not
even noticing the typical problems that people new to C stumble over because
I learned to subconsciously navigate around those issues.

For now I'm planning to develop the C and Zig emulators side by side for
the foreseeable future. This should also allow to extract some useful benchmarking
information to compare performance of C code compiled with Clang versus Zig code
(which at least with the common LLVM backend *should* be identical for the most part).

The long-term goal for the Zig emulator project is essentially to get to the
same state as the C emulators shown here:

[https://floooh.github.io/tiny8bit/](https://floooh.github.io/tiny8bit/)

## Trying out the chipz project

The Github repo is here:

[https://github.com/floooh/chipz](https://github.com/floooh/chipz)

On Mac, Windows or Linux, with a current stable or development Zig version
installed and then just run `zig build --release=fast` (or check the
[README](https://github.com/floooh/chipz/blob/main/README.md) for how to
directly build and start into the emulators:

## Some Emulator Background Info

Both my existing C emulators and the new Zig emulators are based on two 'big
ideas' which differentiates them from most traditional emulators:

- Chip emulators communicate with the outside world exclusively via
  input/output pins, much like real microchips.
- The entire emulation (including the CPU) is cycle-stepped. This
  means that the entire emulation can be suspended and resumed
  at each clock cycle, also right in the middle of a CPU instruction.

Those design ideas allow to 'plug together' a system emulator from chip emulators
much like one would build a real computer prototype on a breadboard, and then
'tick' the system and observe its state after every single clock cycle.

A typical chip emulator (including the CPU) only has one relevant
function which takes an integer of pin bits and returns that same integer
with some bits modified as output. As pseudo code:

```c
    while (true) {
        pins = cpu.tick(pins);
    }
```

To implement a memory access, all necessary information is extracted from the
returned pin mask, and for a memory read the data bus pins are updated
with the memory content.

```c
    while (true) {
        pins = cpu.tick(pins);
        if (pins & MREQ) {
            // a memory read or write access
            const addr = getAddrBus(pins);
            if (pins & WR) {
                // a memory write access
                const data = getDataBus(pins);
                mem[addr] = data;
            } else if (bus & RD) {
                // a memory read access
                const data = mem[addr];
                pins = setDataBus(bus, data);
            }
        }
    }
```

Additional chips in an emulated system also just implement a similar tick
function which takes a 'pin mask' integer as input and returns that same integer
with some pin bits modified.

This 'pure' approach to hardware emulation can be challenging to get fast enough to run
in realtime even on modern computers, since every single clock cycle of every
single chip needs to be emulated on the host CPU in single-threaded code.

For instance to emulate a 4 MHz home computer on a 2 GHz host CPU we have at
most 2GHz / 4Mhz = **500 host CPU clock cycles** to emulate a single emulator
clock cycle, but in reality less than half of that because we need wiggle room to
hit a consistent frame rate - also in debug mode - and we don't want to require
running the emulator on a high end CPU, so a more realistic budget is 100..200
host CPU clock cycles per emulated clock cycle.

Divide that budget further by the work that needs to be performed in
a typical clock cycle: memory access, address decoding, instruction decoding,
updating on-chip counters/timers, video decoding, or even just *checking*
what work actually needs to be performed in the current clock cycle - and it
becomes downright scary.

In the end each chip emulator may have a budget of at most a few dozen host CPU
clock cycles for an emulated clock cycle, and that for very 'branchy' and
unpredictable integer bit twiddling code, and this code can't be multi-threaded
because such code would need to synchronize threads after each emulator clock cycle
(however, SIMD on integer vectors can be useful in specific emulator areas, but
I'm not using that so far).

With all that in mind, the single-thread performance of modern CPUs combined with
modern compilers is **really frigging impressive** (even if it doesn't grow as much
anymore as it used to).

## Use of Special Zig Features

One clear goal I had in mind when starting the Zig emulator project was that
the effort only makes sense when it is more than just a plain port from C.
There's a couple of 'obvious' Zig features to consider:

- Getting rid of preprocessor defines for conditional compilation and 'bit twiddling'
  helper macros. Zig uses comptime code for conditional compilation and offers 'semantic
  inline functions' to replace C helper macros (e.g. 'inline' isn't just an optimization
  hint, but the function will always be inlined, also in debug mode).
- Using narrow integer types for counters and registers. Chips for 8-bit systems
  are full of odd-width counters and registers, like 3-bit counters or 5-bit regsisters.
  Those would map perfectly to Zig's arbitrary-width integer types.
- Using a wide integer type for a shared system bus. If you think of a system emulator
  as chips on a breadboard, and the chip pins connected by wires, you'll need more
  than 64 wires even for simple systems, but also typically not more than 256 wires
  even for complex systems. If each wire is represented by a bit, the shared system
  bus could be represented with a 'wide integer type', for instance u128 or u256.

Interestingly, the more obvious and simple features (inline features and narrow integer
types) may turn out more controversial than the more esoteric feature (representing
a shared system bus with a wide integer).

But one step after another. First let's look at the most important idea that worked
very well: a wide shared system bus and using comptime-code for stamping
out specialized chip emulators and for conditional compilation.

## Zig comptime FTW!

In the C chip emulators, the input/output pin positions are hardwired for each
chip type on a 64-bit unsigned integer.

For instance in the Z80 emulation, the 8-bit data bus might occupy bits 0..7,
the 16-bit address bus bits 8..23, and the remaining 'controls pins' like
`MREQ`, `IORQ`, `WR`, `RD`, `NMI` etc... would occupy bit positions
starting at 24.

In a computer system, some output pins are directly connected to input pins
of other chips, but *which* pins are connected with each other differs
by computer model.

This means that the system tick function (which in turn calls the chip tick
functions) may need to copy pin mask and shuffle bits around because the
output pins of one chip may not be on the same bit position as the connected
input pin of another chip for this specific computer system.

A related problem is when a computer system has multiple chips of the same
type (for instance the KC85/1 has two Z80 PIO chips, or the C64 has two VIAs).
All pins of those identical chip types are on the same pin positions, this means
the tick function needs initialize the pin mask going into such duplicate
chips from sratch and then also needs to extract the output pins from the
result and merge them into other pin mask.

If chip emulators could be comptime-specialized to use different pin bit
positions for different emulated computer systems, a lot of pointless bit
shuffling could be eliminated from a typical system tick function. To make that
idea work we'll also need a wider integer to hold this 'shared system bus' pin
state. IME, 64 bits won't be enough for even simple home computers, but 128
bits should be enough for most.

And this is where Zig's comptime and generics features come in...


## Debug Performance Surprises

TODO

## Integer Expression and Bit Twiddling Woes

TODO

## Things I didn't cover

TODO