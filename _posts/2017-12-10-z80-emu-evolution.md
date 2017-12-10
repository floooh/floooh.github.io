---
layout: post
title: Z80 Emu Evolution
---

...wherein I talk about my latest Z80 emulation adventures.

Before I start, here's a couple of useful links:

- my [YAKC](https://github.com/floooh/yakc) home computer emulator, this contains
the 'old' instruction-stepped Z80 emulator
- my new project [CHIPS](https://github.com/floooh/chips), to move the YAKC
chip emulators into dependency-free C-headers, and to completely overhaul the Z80 emulation and communication between chips
- [z80.info](http://z80.info/) has links to all the other documents I'm mentioning below
- [Ken Shirriff's Blog](http://www.righto.com/) this is a treasure grove of low-level
Z80 information derived directly from die shots
- [Matt Godbolt's Blog](https://xania.org/Emulation) (yes, [the very same](https://godbolt.org/)) this is all about the BBC Micro which had a 6502
CPU, but it gave me a lot of inspiration and ideas for the cycle-ticked Z80 emulator
- and of course [MAME](https://github.com/mamedev/mame), if I couldn't find
any definitive information about hidden or undocumented Z80 features I often used
MAME as 'The Truth' 

## It's this time of the year again...

...where my mind wanders back to emulator coding. I never
got the Amstrad CPC emulation in my [YAKC
emulator](http://floooh.github.io/virtualkc/) good enough to run some more
recent graphics demos correctly, and when I ran the extensive "acid tests" of
the Arnold CPC emulator many of them failed because the emulator timing isn't
precise enough.

So the big topic for this year is to get the CPC emulation right, and for
this I need to start at the bottom by improving the Z80 CPU timing
precision. The current Z80 emulator is only 'instruction-ticked', this means
the emulator is processing a whole CPU instruction atomically, and only
afterward the rest of the system performs the necessary number of ticks to
catch up with the CPU. The main problem is that a single Z80 instruction can
take up to 23 clock cycles to complete and a lot can happen in that time in
the other parts of the emulated system.

On the CPC those 23 cycles are even stretched to 28 cycles because of the CPC's
signature 'wait state injection' to synchronize memory accesses between the
CPU and other chips.

28 clock cycles at 4MHz translates to 7 microseconds, this is a lot of
time considering that the entire visible part of a PAL video scanline is
only 52 microseconds! The more advanced CPC graphics demos are reprogramming
the video hardware several times per scanline, and if this happens a few
pixels too early or late, the whole rendering falls apart and only nonsense
is displayed like this:

![cpc_demo](../../../images/cpc_demo.png)

The reasons why those demos are currently broken most likely isn't the CPU
emulation's fault alone, but without making first sure that the CPU emulation
is correct on a clock-cycle level it is hard to find the problems lurking
in other parts of the emulation.

## The Z80 in a Nutshell

First a quick overview of what makes Z80 emulation a bit 'special'. 

I'll try to keep this short, you can read this older blog post for more
details: [The Amazing Zilog
Z80](http://floooh.github.io/2016/06/15/the-amazing-z80.html).

The key to understand the Z80's quirks (and it is quite full of quirks) is to
know why and how it was created.

The Z80 was built as an Intel i8080 killer:

- it had to be fully backward compatible to the i8080, so it had to
implement the same basic instruction set
- the Z80 had to be more powerful than the i8080 by means of a vastly
expanded instruction set, but within the restrictions of the original i8080,
and the transistor counts that were viable in the mid-70's (the i8080 has
about 4500 transistors, and the Z80 8500, the 6502 has about 3500 in
comparison)
- while the Z80 itself was bigger and more complex than the i8080, it should
trade this added CPU complexity for enabling simpler and cheaper computer
systems by requiring less external circuitry

Since the Zilog engineers had to work within the restrictions imposed by the
i8080 architecture they couldn't design their 'dream CPU' from scratch,
instead they had to invent a lot of clever hacks to build a chip that
appears at least 4x as 'powerful' (in terms of its instruction set) as the
i8080, but with only about twice the transistor count.

Unfortunately these clever hardware hacks make the Z80 much more complicated
to emulate properly than for instance the (very elegantly designed) 6502:

- The instruction decoding (deciding what actions a given opcode
performs) is full of exceptions, the reason for this is mostly that the Zilog
engineers filled whatever 'holes' remained in the original i8080 instruction
set with new Z80 instructions. Any sort of 'orthogonality' mostly went
overboard in the process ([here's a nice overview how a Z80 instruction can
be decoded 'algorithmically'](http://z80.info/decoding.htm))
- The *documented* instruction set is huge for an 8-bit
CPU because the Zilog engineers added 4 whole new 'extended instruction 
ranges' by interpreting the unused i8080 opcodes 0xCB, 0xED, 0xDD and 0xFD as
'prefix bytes', which either modify the behaviour of the following opcode, or add completely new instructions
- The *undocumented* instruction set is even bigger than the documented, this
is because only a fraction of the prefixed instruction ranges are documented,
but *all* instruction slots in those ranges do something when executed
(mostly the same as the corresponding un-prefixed instruction, but taking
longer). All the documented and undocumented instructions add up to nearly
**1700** different cases that must be handled in an emulator.
- The state of the internal 'WZ register' leaks out into the two undocumented
flag bits XF and YF, so a proper emulator must completely emulate this
otherwise completely invisible WZ register. Thankfully some crazy Russian(?)
coders figured out the entire WZ register behaviour [just by observing its effect on 
the 2 undocumented flag
bits](https://raw.githubusercontent.com/floooh/chips/master/info/memptr_eng.txt).
- In order to build Z80 computers from cheap and slow hardware components,
the CPU could be externally throttled by injecting **wait states**, for
instance when talking to slow memory which would take more than one cycle to
complete a read or write operation. Properly implementing wait states make
the instruction cycle count unpredictable from the CPU emulator's point of
view, which makes it harder to optimize for performance. Many home computer
systems (most notoriously my arch-nemesis, the Amstrad CPC) used wait state 
injection to arbitrate memory accesses between CPU and video decoder hardware, 
so it cannot simply be ignored.
- The internal timing *during* instruction execution is full of exceptions as
well. This is important when implementing a cycle-ticked emulator. For
instance an opcode fetch usually takes 4 clock cycles, but in some
instructions it takes 5 or 6. A good overview of how
instructions are split into sub-steps (so called 'machine cycles) and their
execution time [is
here](https://github.com/floooh/chips/blob/master/info/z80ins.txt), but to
get the full story, the cycle timing diagrams in the Z80 manuals must be
consulted as well (for instance in the [MOSTEK Z80
manual](https://github.com/floooh/chips/blob/master/info/z80-mostek.pdf),
which is more comprehensive than the original Zilog manual).

Now how to tackle those challenges:

## Z80 Emulation Strategies

There are mainly 2 'dimensions' to consider when writing a chip emulator:
*state changes* and *timing*. The 'state change dimension' cares about putting
the CPU into a correct 'before and after' state, and the 'timing dimension'
cares about that those changes happen at just the right time.

Another important insight is that only the state that's observable from the
outside is important in the end. In a micro-chip, this observable state
is entirely encoded in the state of the pins that connect the chip to the
rest of the system. The pins are basically the public API of the chip.

### Instruction-Ticked vs Cycle-Ticked

For an **instruction-ticked** emulator the 'observable state' of the CPU
instantly changes from a state before an instruction is executed into a
state after the instruction is executed, and this atomic state change
'magically' costs N clock cycles (but it is unclear where the number of clock
cycles is coming from, it's just looked up in a table).

A **cycle-ticked** emulator splits up the atomic instruction execution into
substeps, and those substeps are visible to the outside.
The overall number of clock cycles for the instruction is derived from the
executed substeps. On the Z80, those substeps are called 'machine cycles',
and each machine cycle takes several clock cycles to complete. For instance:
each instruction starts with an **opcode-fetch** machine cycle, which usually
takes 4 clock cycles to complete, and may be followed by one or several
**memory-read** and **memory-write** machine cycles, each taking 3 clock
cycles (unless stretched by injected wait states). A Z80 emulator may decide
to 'tick' per 'clock cycle', or per 'machine cycle'. Ticking per clock-cycle
is more straight forward but is expensive. Ticking per machine cycle is
faster but more complex because machine cycles have variable length.

A 'tick' in a cycle-ticked emulator usually corresponds to calling a 'tick
callback ' which 'ticks forward' the rest of the emulated system (most
importantly the video and audio systems). Ticking the system forward in such
an 'inside-out' way from the CPU emulator is cheaper than leaving and
entering the CPU emulation for every tick, because then the CPU emulation
would need to store its state after each tick, and restore and continue where
it left off for the next tick. Since the CPU usually has the most complex
internal state in a home computer system, it makes sense to make its life
easier by helping it to carry its internal state from one tick to the next by
'ticking' the other system components from inside the CPU emulation.

### Performance Considerations

The tradeoff between the instruction- or cycle-ticked approach comes down to
performance versus timing precision. An instruction-ticked emulator will
usually be dramatically faster than a cycle-ticked emulator, for instance:

The **instruction-ticked** Z80 CPU emulator in the YAKC emulator runs the
ZEXDOC conformance test in 37.468 seconds on my 2.8GHz Intel i5 MBP. This is
as fast as a 1.2 GHz Z80 would run, or 1.2 billion emulated clock cycles per
second. So each emulated clock cycle only takes between 2 and 3 host CPU
clock cycles! That's pretty much impossible if each emulated Z80 clock cycle
would do actual work on the host CPU, it only works out because the Z80
emulator can cheat and execute instructions atomically.

So let's look at the number of executed instructions instead since that gives
a better idea how well the instruction-ticked emulator performs: the entire
ZEXDOC test executes about 47 billion Z80 clock cycles (46.734.975.782 to be
precise), or about 5.8 billion Z80 instructions (5.764.169.474). So the
average emulated instruction length is 8 Z80-clock-cycles (makes sense
since the shortest Z80 instruction is 4 clocks, and the less frequently used
longest instructions 23 clock cycles).

This is for 37.5 seconds execution time, so that's about 154 million Z80
instructions executed per second (153.842.464). At 2.8 billion host CPU ticks
this means one emulated Z80 instruction takes about 18 host CPU clock ticks
on average for the instruction-ticked emulator. Still not too shabby
considering that each instruction needs to access memory at least once for
fetching the instruction byte. The small memory working set when running the
ZEXDOC test clearly is an advantage here, since the accessed RAM (at most 64 KB),
and the very small Z80 CPU state (less than 32 bytes) should easily fit into
the host CPU caches.

(...man, I surely do hope I didn't mess up the cycle-counting math above, that
would be embarrassing)

Getting this level of performance is impossible-to-pretty-darn-hard for a
**cycle-ticked emulator**. My first naive attempt resulted in about 170
million Z80 clock cycles per second, about 7x slower than the
instruction-ticked version (which means one emulated Z80 instruction takes
about 130 host CPU ticks).

Bummer.

But interestingly this 7x slowdown is pretty similar to the average of 8
clock cycles per Z80 instruction, and the 170 million clock cycles per second
number is pretty close to the about 150 million instructions executed per
second in the instruction-ticked emulator. The cycle-ticked emulator now has
to do something for every **clock cycle**, not only for every
**instruction**, so it makes sense that the work performed per cycle 
aligns with the work per instruction in the old emulator.

After a lot of meditating over compiler output and some (IMHO) quite clever
ideas (more on that later) I managed to speed up the cycle-ticked emulator
about 3 times to nearly half the performance of the instruction ticked
decoder, the current performance stats for the new cycle-ticked emulator
look like this:

The ZEXDOC test finishes in 84.0 seconds (compared to 37.5 seconds for the
instruction ticked version). This means instead of 1.2 GHz, the emulated CPU
runs at 556 MHz, or 556 million emulated ticks per second. The host CPU runs
at 2.8 GHz, so that's about 5 host CPU clock cycles for each emulated Z80
clock cycle. If that sounds a bit unbelievable for a cycle-ticked
emulator, that's because it is ;)

The most important optimization was to actually *not* 'tick' for every single
clock cycle, but instead only perform tick actions when the observable state
of the CPU changes, *or* the CPU might need to react to injected state changes
(such as wait states). More on that later when I'm talking about the 
specific optimizations I implemented.

Before that lets have a look at different instruction decoding strategies:

### Instruction Decoding Strategies

There seem to be 2 popular strategies for vintage CPU emulation:

- **jump tables**: each instruction is its own
little code block jumped to
through a jump table, a 'computed goto', or a giant switch-case statement, this
usually results in a lot of code but has a simple 'execution flow'.
- **algorithmic decoding**: an opcode is split into bit groups, and decoded
'dynamically', this results in much less code, but lots of
branching

Another strategy would be **just-in-time** compilation to translate Z80
instruction sequences into host system instruction sequences, much like
modern compilers for dynamic languages like Javascript do, and theoretically
this should result in the best performance, but a few things common in Z80
programming make this impractical:

- **self-modifying code**: this was extremely popular on the Z80 for code
running in RAM, for instance some code executed before a loop
could modify jump targets for the next loop iterations to reduce the number
of expensive condition checks
- **jumps into the middle of instructions**: another popular method was
to jump into the middle of an instruction, if that 'middle of the instruction'
would form another valid CPU instruction
- **whole-system emulation**: JIT-ing the CPU instructions alone is nearly
useless in an emulated home computer system, since the whole system acted
like a 'super-CPU' which ran all system chips synchronized for each clock
cycle, and things like wait-state injection make the CPU timing unpredictable, and
require to consider the state of the entire system on each clock tick.

So currently I don't see how JIT-ing could work with reasonable programming 
effort and performance for emulating entire 8-bit computer systems.

Back to the 'giant switch/case' vs 'algorithmic' decoding. Over time I
went back and forth at least 3 times between the two strategies. I usually
start with the algorithmic decoder, hoping that the small amount of code
would make better use of the instruction cache, but then inevitably find out
that the brute force approach of a giant switch/case would run at least twice
as fast, but I still weep because of the huge amount of code required for
a switch/case decoder.

The specific problem with algorithmic decoding of Z80 instructions is the many
special cases, which require a lot of indirections for accessing registers and
a lot of branching. For a 6502 emulator the algorithmic approach makes a lot
more sense because the 6502 instruction set is much more streamlined.

My most recent approach for the Z80 is thus a hybrid decoder. The opcode 
is still going through a giant switch-case, but some opcode ranges which are
very 'uniform' use algorithmic decoding. For instance [here is the current Z80
decoder](https://github.com/floooh/chips/blob/090828dc04b508c5dd437546989e87433ac0370a/chips/_z80_opcodes.h). The entire [0xCB prefix range is decoded dynamically](https://github.com/floooh/chips/blob/090828dc04b508c5dd437546989e87433ac0370a/chips/_z80_opcodes.h#L260). This little change reduces the amount of x86 code
by about half without compromising overall performance (from 190KBytes to about 100KBytes x86-64 binary code at same ZEXDOC performance).

That C source code is generated by [this python script](https://github.com/floooh/chips/blob/master/codegen/z80_opcodes.py), which
essentially implements [this decoding recipe](http://www.z80.info/decoding.htm).

Another interesting side effect of the code-generation approach is that
the manually written [z80.h
header](https://github.com/floooh/chips/blob/090828dc04b508c5dd437546989e87433ac0370a/chips/z80.h)
is nearly empty. All the interesting stuff, including interrupt handling is in the single,
generated **z80_step()** function.

## The CHIPS Project

Ok, on to the last part, implementation details of the new Z80 emulator.

I have started a new github project called CHIPS here:

[https://github.com/floooh/chips](https://github.com/floooh/chips)

The 3 main ideas behind the project are:

- provide a collection of header-only, dependency-free 8-bit chip emulators written in C
- rewrite the Z80 emulation from 'instruction ticked' to 'cycle ticked'
- rewrite the 'chip communication protocol' from specialized callbacks to 
'chip pin bitmasks'

## Communicating through Pins

The whole idea of using a bitmask of pin states to handle communication is
the most interesting 'bit' of the new emulator, and only makes sense with
a cycle-ticked emulation.

A Z80 has 40 pins, the important ones are:

- **A0..A15 (out)**: this is the 16 bit address-bus, used for addressing 
64 KBytes of memory or as port number for communicating with other chips
and hardware devices
- **D0..D7 (in/out)**: the 8-bit data bus, the address bus says 'where'
to read or write something, and the data bus says 'what'
- **MREQ (out)**: the 'memory request' pin is active when the CPU wants to
perform a memory access
- **IORQ (out)**: likewise the 'I/O request' pin is active when the CPU wants
to perform an I/O device access (via the special IN/OUT instructions of the Z80)
- **RD (out)**: the 'read' pin is used together with the MREQ or IORQ to identify
a memory-read or IO-device-input operation
- **WR (out)**: ...and this is for the opposite direction, a memory-write or IO-device-output
- **M1 (out)**: 'machine cycle one', this pin is active during an opcode fetch
machine cycle and can be used to differentiate an opcode fetch from a normal
memory read operation
- **WAIT (in)**: this pin is set to active by the 'system' to inject a wait state
into the CPU, the CPU will only check this pin during a read or write operation
- **INT (in)**: this pin is set to active by the 'system' to initiate an interrupt-
request cycle
- **RESET (in)**: this pin is set to active by the 'system' to perform a system reset

These are the important pins for most home computer emulators, the remaining
pins (NMI, RFSH, BUSRQ, BUSAK) are currently not used.

When the CPU executes instructions, it performs a sequence of operations called
'machine cycles', where each machine-cycle is usually 4 or 3 clock cycles long
(but as mentioned above, there are a lot exceptions to the rule).

The machine cycle types relevant for home computer emulation are:

### Opcode Fetch 

This reads the next opcode byte from memory, and decodes it. Usually this
takes 4 clock cycles, the first 2 clock cycles perform a memory read to get
the next opcode byte, and the last 2 clock cycles are used for decoding the
opcode byte. Since the address bus is unoccupied during those final 2 ticks,
it is used for a 'memory refresh operation' (which I'm currently ignoring in
the emulator).

The CPU pins look like this for each clock cycle (XX means 'active'):

```
  M1  MREQ IORQ  RD   WR   RFSH | ADDR   DATA   tick:
| XX | XX |    | XX |    |      |  PC  |  ??  | 1
| XX | XX |    | XX |    |      |  PC  |  23  | 2
|    | XX |    |    |    |  XX  |  IR  |  ??  | 3
|    |    |    |    |    |  XX  |  IR  |  ??  | 4
```

In the first tick, the CPU puts the PC register (program counter)
on the address bus, and set the M1|MREQ|RD pins, the MREQ|RD cause
a memory read, and the resulting byte will be placed on the
data bus pins in the next tick. In the next two ticks the 
instruction decoder decodes the loaded opcode byte and decides
what to do next. In those two cycles the address and data bus would
be idle and are used for the memory refresh operation.

During the second tick (and only then) the CPU will check the state
of the WAIT pin, and will start inserting 'wait states' until the
WAIT pin goes inactive.

### Memory Read and Write 

Many instructions require reading or writing memory. In this case,
Memory Read (MR) and/or Memory Write (MW) machine cycles are following after
the Opcode Fetch machine cycle. MR and MW cycles are generally 3 clock
cycles long unless stretched by wait states.

The pin pattern for a Memory Read looks like this:
```
  M1  MREQ IORQ  RD   WR   ADDR   DATA   tick:
|    | XX |    | XX |    | addr |  ??  | 1
|    | XX |    | XX |    | addr |  ??  | 2
|    |    |    |    |    | addr |  45  | 3
```

And here for a Memory Write:
```
  M1  MREQ IORQ  RD   WR   ADDR   DATA   tick:
|    | XX |    |    |    | addr |  67  | 1
|    | XX |    |    | XX | addr |  67  | 2
|    |    |    |    |    | addr |  67  | 3
```

Again, the WAIT pin is only sampled by the CPU during the second
clock tick.

### Input/Output Cycle

These are similar to Memory Read and Memory Write machine cycles,
but with active IORQ pin instead of the MREQ pin, and they are
1 clock cycle longer (4 instead of 3) to give the IO device a bit
more time to react.

### Interrupt Request/Acknowledge Cycle

This is an optional machine cycle which will be inserted
after a complete instruction when the INT pin is active.
It looks a lot like an opcode fetch stretched to 5 cycles
but with the MREQ|RD pins off, and instead M1 and IORQ
active. Interrupt controllers check these pins and
write an interrupt vector to the data bus. But the whole
interrupt handling process goes a bit too far for this blog
post, so I'll skip this topic :)

### Machine Cycles as Instruction Building Blocks

With the basic machine cycles described above, the instruction
cycle counts suddenly make a lot more sense.

For instance the instruction ```ADD A,B``` doesn't read or write
memory except for the initial opcode fetch, and a normal opcode
fetch takes 4 clock cycles, and indeed the complete cycle count
for the ```ADD A,r``` instruction is 4.

The instruction ```ADD A,n``` on the other hand needs to read
the immediate byte operand from memory, so it needs one 
opcode fetch (4 clock cycles) and one memory read (3 clock cycles),
resulting in 7 clock cycles for the whole instruction.

This works well except for a few annoying special cases which require
'filler clock cycles', [as can be gathered from this table](https://github.com/floooh/chips/blob/master/info/z80ins.txt).

## The Tick Callback

As mentioned above, the new CPU emulator only has a single callback
function, instead of many specialized callbacks. This single callback
function has an interesting history of optimizations.

The very first version looked like this:

```c
void tick(z80* cpu) {
    if (cpu->pins & Z80_MREQ) {
        /* a memory request */
        if (cpu->pins & Z80_RD) {
            /* a memory read */
            cpu->data = mem[cpu->addr];
        }
        else if (cpu->pins & Z80_WR) {
            /* a memory write */
            mem[cpu->addr] = cpu->data;
        }
    }
    else if (cpu->pins & Z80_IORQ) {
        /* an I/O request */
        ...
    }
}
```
This was called for every clock tick, 4 times for simple instructions
and 23 times for the most complex instructions.

And it resulted in terrible performance (170 million emulated clock ticks
per second, compared to 1.2 billion for the old instruction-ticked 
emulator).

The code which called the tick callback up in the instruction decoder
looked simple enough:

```c
    ...;cpu->tick(cpu);ticks++;...
```

Sifting through the generated assembly code I noticed that the tick
callback function pointer was loaded from memory
*every single time* before invoking the tick callback, even though the tick
callback didn't actually change. But the compiler doesn't know that, it has
to assume that the tick callback function might change the function pointer,
so it has to reload it after each tick (that was my interpretation at least).

A simple change to put the tick function pointer into a local variable
improved the performance very noticeably:

```c
//...at the start of the instruction decoder:
const tick_callback tick = c->tick;
//...and later:
...;tick(cpu);ticks++;
```

This simple innocent change provides an important hint to the compiler, since
the local variable isn't visible to the tick callback, it's guaranteed not
to change. The result was that the compiler now kept the callback pointer
in a register in the entire instruction decoder.

But there was still a lot of memory accesses happening around the tick callback,
so the next step was to only communicate the 'public CPU state' into
the tick callback, which now looked like this:

```c
uint64_t tick(uint64_t pins) {
    if (pins & Z80_MREQ) {
        if (pins & Z80_RD) {
            Z80_SET_DATA(pins, mem[Z80_ADDR(pins)]);
        }
        else if (pins & Z80_WR) {
            mem[Z80_ADDR(pins)] = Z80_DATA(pins);
        }
    }
    else if (pins & Z80_IORQ) {
        ...
    }
    return pins;
}
```

That's right, the entire public CPU state (the 40 CPU pins) are packed into a
single 64-bit integer, including the address- and data-bus! The tick callback
shouldn't be concerned about the internal CPU state (all the register
values), it only needs to inspect the CPU pins, optionally change the pins
mask, and return it to the instruction decoder!

This was the first big performance breakthrough, since it reduced the number
of memory accesses in the CPU decoder and tick callback dramatically and nearly
doubled the performance to about 300 million emulated clock ticks per
second!

For the (so far) final big performance jump I cheated a bit. Instead of
calling the tick callback for every single clock cycle, I'm now allowing
to merge several ticks into a single callback invocation, so the latest
version of the callback looks like this:

```c
uint64_t tick(int num_ticks, uint64_t pins) {
    ...
}
```

The instruction decoder will now merge several ticks into one callback
invocation 'if nothing interesting happens' between multiple ticks. This is
also the reason why I'm ignoring the 2 memory refresh cycles during the
opcode fetch machine cycle. With this little bit of cheating I can handle the
entire 4 cycle opcode fetch with a single invocation of the tick callback. 

If a wait state is injected the tick callback will still be called for single
ticks until the wait state is cleared, but in most other situations multiple
ticks can be merged into a single callback invocation.

This again nearly doubled the performance to the 550 million emulated ticks
per second I'm seeing now, and I'll keep it at that for now (however one
thing I will try soon-ish is [Insomniac's cache-simulator](https://deplinenoise.files.wordpress.com/2017/03/cachesimstyled.pdf), this
should yield interesting results).

It is important to note that the performance is **extremely dependent** on
the tick callback. Even small and innocent looking changes in the tick
callback can cut the performance in half, once a whole computer system
emulation is running inside the tick callback it will most likely dominate
the performance, and the CPU will only play a minor role.

## Communicating with other Chips

So far I have added 2 more members of the Z80 chip family, the
Z80 PIO (Parallel Input Output), and the Z80 CTC (Counter/Timer Channels).

These have a number of pins which are usually directly connected to the
CPU pins of the same name:

- **D0..D7**: the data bus is directly shared with the CPU and other chips
- **IORQ**: used to detect whether the CPU wants to perform an I/O operation
- **RD**: used to detect whether this is a read or write operation (curiously
there is no WR pin on the CTC or PIO)
- **INT**: both the PIO and CTC can act as interrupt controllers, so they
connect directly to the CPU's INT pin (again I'm skipping the interrupt
handling details in this blog post)
- **M1**: the M1 pin is used together with the IORQ pin to detect an interrupt
acknowledge cycle (skip!)

The 64-bit pin masks of the CTC and PIO are arranged so that the shared pins
with the CPU are on the same bit locations, that way no expensive bit shuffling
operations need to happen when communicating state between the CPU and other
chips.

## What's Next

I will now start to replace the YAKC chip emulators with the new
header-only C versions, and convert the other 'old style' chip emulators
to header-only and C as needed. This will also result in some improvements
in the non-CPC emulators, for instance both the KC85 and Z9001/KC87 computers
are injecting wait states when accessing video memory, the current emulators
ignore this.

Finally I'll tackle the CPC emulator again with the goal to get the
Arnold Acid Tests running, and the graphics demos that are currently broken.

And finally a glimpse of what might come in a more distant future... The
basic idea to pack the CPU pins into a single 64-bit integer leads to a more
crazy idea... what if we could pack the entire dynamic state of a complete
home computer (minus the RAM of course) into one or very few SIMD registers
(split into many 8- and 16-bit integers), and 'tick' this super-wide SIMD
register through a sequence of shift-, mask- and ALU-instructions to step the
entire computer system forward at once. Most dynamic state in an 8-bit home
computer actually consists of [counters that tick other
counters](http://floooh.github.io/2017/02/04/yack-counters.html), only the
the more complex CPU emulation might sink this idea.

But first things first, and that's a correct and fast CPC emulation :)
