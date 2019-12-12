---
layout: post
title: More Emulation Adventures (2019 Edition)
---

TL;DR: A new cycle-stepped 6502 emulator, and how I got there.

I wrote a new version of my 6502/6510 emulator in the last weeks which can
be stepped forward in clock cycles instead of full instructions.

The actual work didn't take a couple of weeks, more like a few evenings and
a weekend, because the new emulator is more or less just a different output 
of the code-generation python script which I keep mutating for each new
emulator version.

But I got a bit sidetracked by a somewhat related project that just had to
be done once I was diving back into 6502 emulation:

https://floooh.github.io/visual6502remix/

But this project is (maybe) the topic for another blog post.

Before I'm getting to the cycle-stepped 6502 emulator, a little detour
into 8-bit CPUs and CPU emulators in general:

## What CPUs actually do

The general job of a CPU (no matter if old or new) is quite simple: given
some memory filled with instructions and data, process instructions one after
another to (ultimately) change some of the memory content into something
else, and repeat this process forever. Everything else in a typical computer
just exists to turn some of those memory changes into pretty pixels and sounds.

The processing of a single instruction can be broken down into several steps:

- fetch the instruction opcode from memory (on 8-bit CPUs this is the
first (and sometimes only) byte of an instruction)
- 'decode' this opcode to decide what actions must be taken to
'execute' the instruction, such as:
- load additional data from memory into the CPU
- do some specific operation (arithmetics, bit twiddling, etc)
- store data from the CPU back to memory
- repeat those steps for the next instruction

On top of this simple 'core execution loop' which allows to run through simple
sequences of instructions exist some higher level mechanisms which all deal
with control flow:

- **branches**: Jump to a different memory location and continue executing instructions from there, this is like a goto in higher level languages
- **conditional branches**: Based on the result of a previous computation, either jump to
a different memory location, or continue with the instruction directly following the
branch instruction. Conditional branches are the assembly-level building block of the higher-level
language constructs like if(), for() and while() 
- **subroutine calls**: Store the current execution location on the 'stack'
(a special area in memory set aside for storing temporary values), jump to a
different memory location (the subroutine), and at the end of that subroutine
(indicated by a special 'return' instruction), load the stored execution
location from the stack and continue execution right after the original
subroutine call. This is like a function call in a high level language.
- **interrupts**: Interrupts are like subroutine calls except they are
initiated by a hardware event (such as a hardware timer/counter reaching
zero, a key being pressed, or the video signal reaching a specific raster
line). The CPU stops whatever it is currently doing, jumps into a special
'interrupt service routine', and at the end of the service routine, continues
with whatever it did before.

CPUs usually have a few special registers to deal with control flow:

- The current execution location (where the next instruction is loaded from)
is stored in the *Program Counter* (PC) register.
- The current stack location is stored in the *Stack Pointer* (S or SP) register.
- A *Flags or Status Register* (for some mysterious reason called 'P' on the 6502)
which stores (mostly) information about the result of previous arithmetic
operations to be used for conditional branching. For instance: if the last
instruction subtracted two numbers and the result is 0, the Flags Register
will have the *Zero Flag* bit set. The conditional branch
instructions BEQ (branch when equal) and BNE (branch when not equal) look at
the current state of the Zero Flag to decide whether to branch or not.

It's interesting to note that the branch instructions of a CPU are essentially
normal load instructions for the PC register, since that's what they do: loading
a new value into the PC register so that the next instruction is fetched
from a different memory location.

## The components of an 8-bit CPU

8-bit CPUs like the Z80 or 6502 are fairly simple from today's point
of few: a few thousand transistors arranged in a few logic blocks:

A **Register Bank** with somewhere between a handful and a dozen 8- and 16-bit
registers (the 6502 has about 56 bits worth of 'register state', while the
Z80 has over 200 bits)

An **ALU** (Arithmetic Logic Unit) which implements integer operations like
addition, subtraction, comparison (usually by doing a subtraction and
dropping the result), bit shifting/rotation, and the logic operations AND, OR
and XOR.

The **Instruction Decoder**. This is where all the 'magic' happens. The
instruction decoder takes an instruction opcode as input, and in the
following cycles runs through a little instruction-specific hardwired program
where each program step describes how the other components
of the CPU (mainly the register bank and ALU) need to be wired together and
configured to execute the current instruction substep. 

How exactly the instruction decoder unit is implemented differs quite a lot
between CPUs, but IMHO the general mental model that each instruction is a
small hardwired 'micro-program' of substeps which reconfigures the data flow
within the CPU, and the current action of the ALU is (AFAIK) valid for all
popular 8-bit CPUs, no matter how the decoding process actually happens in
detail on a specific CPU.

And finally there's the 'public API' of the CPU, the **input/output pins**
which connect the CPU guts to the outside world.

Most (all?) popular 8-bit CPUs look fairly similar from the outside:

- 16 address bus output pins, for 64 KBytes of directly addressable memory
- 8 data bus in/out pins, for reading or writing one data byte at a time
- a handful of control- and status-pins, these differ between
CPUs, but common and important pins are:
    - one or two RW (Read/Write) output pins to indicate to the outside world if the CPU
      wants to read data from memory (with the data bus pins acting as inputs), 
      or write data into memory (data pins as outputs) at the
      location indicated by the address bus pins
    - IRQ and NMI: These are input pins to request a "maskable" (IRQ) or
      non-maskable (NMI) interrupt.
    - RES: an input pin used to reset the CPU, usually together with the
      whole computer system, this puts the CPU into a defined starting
      state

Of course the CPU is only one part of a computer system. At least some memory
is needed to store instructions and data. And to be any useful
there must be a way to get data into and out of the computer (a keyboard, 
joystick, display, loud speakers, and a tape- or floppy-drive). These 
devices are usually not controlled directly by the CPU, but by additional
helper chips, for instance in the C64:

- 2 CIA chips to control the keyboard, joysticks, and tape drive
- the SID chip to generate audio
- and the VIC-II chip to generate the video output

It must be noted that the C64 is definitely on the extreme side for 8-bit
computers when it comes to hardware complexity. Most other 8-bit home
computers had a much simpler hardware configuration and were made of 
fewer and simpler chips.

All these chips and the CPU are connected to the same shared address- and data-bus,
and there's some additional 'address decoder logic' (and clever engineering
in general) needed so that all those components don't generate a mess on the
shared busses, but diving in there goes a bit too far for this blog post :)

Back to emulators:

## How CPU emulators work

While all CPU emulators must somehow implement the tasks of real CPUs outlined
above, there are quite a few different approaches how they reach that goal.

The range basically goes from "fast but sloppy" to "accurate but slow". Where
on this range a CPU emulators lives depends mainly on the computer system
the emulator is used for, and the software that needs to work:

- Complex general-purpose home computers like the C64 and Amstrad CPC with an
active and recent demo scene need a very high level of accuracy down to the
clock cycle, because a lot of frighteningly smart demo-sceners are exploring (and
exploiting) every little quirk of the hardware, far beyond what the original
hardware designers could have imagined, let alone what's written in the
system specifications and chip manuals.

- The other extreme are purpose-built arcade machines which only need to run
a single game. In this case, the emulation only needs to be 'good enough' to
run one specific game on one specific hardware, and a lot of shortcuts can be
taken as long as the game looks and feels correct.

Here's the evolution from "fast but sloppy" to "slow but accurate" CPU
emulators that I've gone through, I'm using the words "stepped" vs "ticked"
in a very specific way:

- "stepped" means how the CPU emulator can be "stepped forward" from the outside
by calling a function to bring the emulator from a "before state" into an "after
state"
- "ticked" means how the *entire emulated system* is brought from a "before state"
into an "after state"

### Instruction-Stepped and Instruction-Ticked

This was the first naive implementation when I started writing emulators, a
Z80 emulator which could only be stepped forward and inspected
for complete instructions.

The emulator had specialized callback functions for memory accesses, and
the Z80's special IO operations, but everything else happening "inside"
an instruction was completely opaque from the outside.

This means that an emulated computer system using such a CPU emulator could only
"catch up" with the CPU after a full instruction was executed, which on the
Z80 can take anywhere between 4 and 23 clock cycles (or even more when an
interrupt is handled).

This worked fine for simple computer systems which didn't need accurate
emulation, like the East German Z- and KC-computers. But 23..30 clock cycles
at 1 MHz is almost half of a video scanline, and once I moved to more complex systems like the Amstrad CPC
it became necessary to know when exactly within an instruction a memory
read or write access happened. The Amstrad CPC's video system can be
reprogrammed at any time, for instance in the middle of a scanline, and when
such a change happens at the wrong clock cycle within a scanline, things start to
look wrong, from subtle things like a few missing pixels or wrong colors here
and there to a completely garbled image.

### Instruction-Stepped and Cycle-Ticked

The next step was to replace the specialized IO and memory access callbacks
with a single universal 'tick callback', and to call this from within the
CPU emulation for each clock cycle of an emulated instruction.

From the outside, it's still only possible to execute instructions as a whole.
So calling the emulators 'execute' function once will run through the entire
next instruction, calling the tick callback back several times in turn once for each
clock cycle. It's not possible to tell the CPU to only execute one tick,
or suspend the CPU in the middle of an instruction.

With this approach, the CPU takes a special, central place in an emulated
system. The CPU is essentially the controller which calls one central 'tick
callback', and the entire remaining system is emulated in this tick callback.
This allows a perfectly cycle accurate system emulation, and as far as I'm
aware, this approach is used in most 'serious' emulators, and to be honest,
the reasons to use an even finer-grained approach are a bit esoteric:

### Cycle-Stepped and Cycle-Ticked

This is where I'm currently at with the 6502 emulation (but not yet the Z80 emulation).

Instead of an "exec" function which executes an entire instruction, there's now
a "tick" function which only executes one tick of an instruction and returns
to the caller. With this approach a 'tick callback' is no longer needed, and
the CPU has lost it's special 'controller status' in an emulated system.

Instead the CPU is now just one chip among many, ticked forward like all the
other chips in the system. The 'system tick' function has inverted its role.
Instead of being called from inside the CPU emulation, the CPU emulation is
now called from inside the system tick function. This makes the entire
emulator code more straightforward and flexible. It's now 'natural' and
trivial to tick the entire system forward in single clock cycles (yes, the
C64 emulation now has a c64_tick() function).

Being able to tick the entire system forward in clock cycles is very useful
for debugging. So far, the step-debugger could only step through CPU instructions.
That's good enough for debugging CPU code, but not great for debugging the VIC-II
or CIAs. Each debug step would skip over several clock cycles depending on what
CPU instruction is currently executed. But now that the CPU can be cycle-stepped,
implementing a proper 'cycle-step-debugger' is fairly trivial.

Another situation where a cycle-stepped CPU is useful is multi-processor systems,
which actually wasn't that unusual: the Commodore 128 had both a 6502 and Z80 CPU,
some arcade machines had sound and gameplay logic running on different CPUs, 
and even attaching a floppy drive to a C64 turns this into a multiprocessor
system, because the floppy drive was its own computer with its own 6502 CPU.

With a cycle-stepped CPU, it's much more straightforward now to write
emulators for such multi-CPU systems. If required, completely unrelated
systems can now run cycle-synchronized without weird hacks in the tick
callbacks or variable-length 'time slices'.

## Implementation Details


...



Another interesting aspect in that area of 'precision versus performance' is that
CPU emulators rarely dominate system emulation performance. For instance in the
[C64 emulator](https://floooh.github.io/tiny8bit/c64.html) (which admittedly
is an extreme case), the CPU emulation cost is only about 3..5% of the overall
system emulation.

By far the biggest cost (50..60%) is in the video emulation (the VIC-II chip
emulation), and the rest is split fairly evenly between the two CIAs and the
SID emulation.

It's somewhat surprising (at least it was to me) that CPU cost is so tiny compared to
the other components, but it becomes clear when looking at what the CPU emulation
actually needs to do per emulated clock cycle vs what the VIC-II needs to do:

One clock cycle on the CPU roughly means doing either: a memory read, a memory
write or a simple arithmetic or logic operations (which are a bit more 
expensive than they sound because status bits must be updated). But on average
it's still just a handful host-system instructions per emulated clock cycle.

Here's what a VIC-II must do per 1 MHz clock cycle: determine the color
of 8 pixels, depending on display mode, position on screen, the
state of various VIC-II registers, and finally 8 sprite units which may overlap
the current pixel. And all that may change from one clock cycle to the next. 
Emulating a single VIC-II clock cycle may require dozens to hundreds of
host system instructions.

The performance range for CPU emulations goes from roughly a hundred times slower
than a real 6502 for the 'perfectly accurate' case of a transistor-level
simulation like [perfect6502](https://github.com/mist64/perfect6502) to 
somewhere between a hundred to a thousand times faster for more traditional
emulators depending on where they are on the "sloppiness vs accuracy" scale.

But as I said above, whether a CPU emulation is a hundred times faster or a
thousand times faster than the real thing isn't all that important in the big
picture of a complete home computer emulator. In the case of my C64
emulation, speeding up the CPU emulation by ten times wouldn't make much of a
difference for the entire system emulation. Reducing the execution time of a
component from 3% to 0.3% would hardly be noticeable.

This was also the main reason why I decided to go in the opposite direction
and trade some performance for the ability to step the CPU emulation forward
by single clock ticks. I knew it would be slower from the start. And it wouldn't
even be more accurate out of the box, it would just make it *easier* to be
more accurate, and a cycle-stepped emulator would open up some more
"opportunities" in the future.













## Current state of the 6502 and C64 emulators 

The new 6502 emulator can be seen in action in the C64 and Acorn Atom
emulators here:

- https://floooh.github.io/tiny8bit/c64-ui.html
- https://floooh.github.io/tiny8bit/atom.html

6502 source code is here:

- https://github.com/floooh/chips/blob/master/chips/m6502.h

Code-generated from these two files:

- https://github.com/floooh/chips/blob/master/codegen/m6502.template.h
- https://github.com/floooh/chips/blob/master/codegen/m6502_gen.py

The C64 system emulator is here:

- https://github.com/floooh/chips/blob/master/systems/c64.h

There's also quite a few new C64 demo-scene demos on the [Tiny Emulators main
page](https://floooh.github.io/tiny8bit/). Those demos give a good impression
about the overall emulation quality, since they often use the hardware in
interesting ways and have strict timing requirements.

The C64 emulation is now "pretty good but still far from perfect", while the
CPU and [CIA](https://en.wikipedia.org/wiki/MOS_Technology_CIA) accuracy are much better now, the
[VIC-II](https://en.wikipedia.org/wiki/MOS_Technology_VIC-II) emulation leaves a lot to
be desired (this will be my next focus for the C64 emulation, but I'll most
likely spend a bit of time with something else).

Here are some C64 demos which still show various rendering artefacts:

- [Party Horse by Booze Design](https://floooh.github.io/tiny8bit/c64.html?file=c64/party_horse.prg)
- [Summer of 64 by Offence](https://floooh.github.io/tiny8bit/c64.html?file=c64/summerof64.prg)
- [One-Der by Oxyron](https://floooh.github.io/tiny8bit/c64.html?file=c64/oneder_oxyron.prg)

Some other demos even get stuck, which is most likely related to the VIC-II
raster interrupt firing at the wrong time, but as I said, the VIC-II
emulation quality will be my next focus on the C64.

The CPU and CIA emulation are nearly (but not quite) perfect now, there's one
problem related to the CIA-2 and non-maskable interrupts which I wasn't able
to track down yet.

Here's an overview of the conformance tests I'm currently using, and their
results:

All **NEStest CPU tests** are succeeding, these are fairly "forgiving" high level
tests which only test the correct behaviour and cycle duration
of documented instructions without the BCD mode.

All **Wolfgang Lorenz tests** are succeeding, except:
    - the *nmi* test has one failed subtest
    - the four CIA realtime clock related tests are failing because my current
    CIA emulation doesn't implement the realtime clock

The Wolfgang Lorenz test suite covers things like:
    - all instructions doing the "right thing" and taking the right
      number of clock cycles, including all undocumented and unintended
      instructions, and the BCD arithmetic mode
    - various 'branch quirks' when branches go to the same or a different 
      256 byte page
    - correct timing and behaviour of interrupts, including various
      interrupt related quirks both in the CPU and CIAs (like NMI
      hijacking, both CIAs requesting interrupts at the same time,
      various delays when reading and writing CIA registers etc)
    - various behaviour tests of the 6510's IO port at address 0 and 1
    - and some other even more obscure stuff

The big area that's not covered by the Wolfgang Lorenz test suite is the
VIC-II, for this I've started to use tests from the [VICE-emulator test
bench](https://vice-emu.pokefinder.org/index.php/Testbench) now. But these
tests are "red" all over the place at the moment, and it's not worth yet
writing about them :)

A new automated testing method I'm using for the cycle-stepped 6502 emulation
is [perfect6502](https://github.com/mist64/perfect6502) this is a C port
of the transistor-level simulation from [visual6502](http://visual6502.org/).

Perfect6502 allows me to compare the state of my 6502 emulation against the
'real thing' after each clock cycle, instead of only checking the result of
complete instructions. The current test isn't complete, it only checks the
documented instructions, and some specific interrupt request situations. But
the point of this test is that it is trivial now to create automated tests
which allow checking of specific situations against the real transistor-level
simulation.

