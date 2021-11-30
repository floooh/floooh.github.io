---
layout: post
title: Getting into way too much detail with the Z80 netlist simulation
---

This is part one of a two-part series about a new cycle-stepped Z80 emulator
I wrote recently. In the first post I'll mainly take a lot at the oddities and
irregularities in the Z80 instruction set with the help of the Z80 netlist
simulation from [visual6502.org](http://www.visual6502.org/JSSim/expert-z80.html)
which I integrated into my own 'remix' before starting to work on the actual
CPU emulator:

[https://floooh.github.io/visualz80remix/](http://www.visual6502.org/JSSim/expert-z80.html)

The 'remix' has some usability advantages over the original:

- rendering and UI performance is much improved via WASM, WebGL and Dear ImGui
- an integrated assembler simplifies program input
- a tracelog window which shows more information and allows to 'rewind' the simulation

There's a lot more information in the project readme here: [https://github.com/floooh/v6502r](https://github.com/floooh/v6502r).

A word of warning though, the Z80 netlist from visual6502 has some subtle
differences in undocumented behaviour from what's known about original Z80's
(more on this toward the end of this blog post). My guess is that the netlist
has been created from a Z8400 because of those two details found on the die:

![z80_detail]({{ site.url }}/images/z80_detail.jpg)

![z80_detail_2]({{ site.urk }}/images/z80_detail_2.jpg)

...at least this might explain why the netlist doesn't suffer from the six 'traps' that were 
placed on the original Z80 to hinder reverse engineering efforts. By the time the Z8400 was
created the Z80 had already been widely cloned so any additional effort to make reverse
engineering harder would have been pointless.

Anyway, back to the actual topic of the blog post:

The Z80 has probably the most messy and irregular instruction set of the popular
8-bit CPUs (certainly when compared to the MOS 6502 or Motorola 6800). The reason
is that the Z80 had to be binary compatible with the Intel 8080. While the 8080
has a reasonably clean and structured instruction set, the only way for the Z80
to add new instructions while remaining 8080-compatible was to fill the 'gaps' 
in the 8080 instruction set.

This is why the Z80 instruction set looks clean and structured only in some places
(mainly those inherited from the 8080), while at the same time being peppered
with seemingly random new Z80-specific instructions. This was the right approach
to create an "8080 killer", but nearly half a century later it makes life a lot
harder for emulator authors :)

## The Z80 instruction byte structure

Like on the Intel 8080, instruction are made up of one or multiple bytes, where
the first byte is always the opcode byte.

This simple rule is also true for the Z80-specific 'prefixed instructions', which
superficially seem to have two opcode bytes. But as we'll see later, instruction
prefixes are actually complete instructions on their own which just influence how
the following opcode byte is decoded. 

Prefixes aside, there are only three basic instruction 'shapes':

Just the opcode byte (for example **NOP**, **LD A,B**, **RET**):
```
┏━━━━━━━━┓
┃ OPCODE ┃
┗━━━━━━━━┛
    C9      => RET
```

An opcode byte followed by an 'immediate value' byte (for example **LD A,n**, **ADD n**, **JR d**):
```
┏━━━━━━━━┳━━━━━━┓
┃ OPCODE ┃ IMM8 ┃
┗━━━━━━━━┻━━━━━━┛
    3E      11      => LD A,11h
```

An opcode byte followed by two immediate value bytes, used as a 16-bit value (for example
**LD HL,nnnn**, **CALL nn**):
```
┏━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┓
┃ OPCODE ┃ IMM16LO ┃ IMM16HI ┃
┗━━━━━━━━┻━━━━━━━━━┻━━━━━━━━━┛
    21       34         12      => LD HL,1234h
```

Instructions that have been 'modified' by the DD or FD prefix may come in two additional
shapes where an offset byte is inserted after the opcode and up to one immediate value byte
(this is the 'd' in **(IX+d)** or **(IY+d)**). There is no instruction where the d-offset byte
is followed by an 16-bit immediate value (why that's the case will become clear later in the
section about the DD/FD prefixes):
```
┏━━━━━━━━┓┏━━━━━━━━┳━━━━━━━┓
┃ PREFIX ┃┃ OPCODE ┃ DIMM8 ┃
┗━━━━━━━━┛┗━━━━━━━━┻━━━━━━━┛
    DD        86      03        => ADD A,(IX+3)

┏━━━━━━━━┓┏━━━━━━━━┳━━━━━━━┳━━━━━━┓
┃ PREFIX ┃┃ OPCODE ┃ DIMM8 ┃ IMM8 ┃
┗━━━━━━━━┛┗━━━━━━━━┻━━━━━━━┻━━━━━━┛
    FD        36      03      11        => LD (IY+3),11h
```

The last special instruction shape looks extremely weird because at first glance it doesn't
fit into the above patterns at all. In 'double-prefixed' instructions (like **SET 1,(IX+3)**)
the d-offset and 'actual' opcode seems to have switched places!
```
┏━━━━━━━━┓┏━━━━━━━━┓┏━━━━━━━┳━━━━━━━━┓
┃ PREFIX ┃┃ PREFIX ┃┃ DIMM8 ┃ OPCODE ┃
┗━━━━━━━━┛┗━━━━━━━━┛┗━━━━━━━┻━━━━━━━━┛
    DD        CB       03       CE      => SET 1,(IX+3)
```
Those double-prefixed instructions are definitely the biggest oddity in the
whole instruction set. At least this specific oddity (d-offset before the
opcode) will be explained later in the dedicated DD/FD+CB prefix section.

Then it will also become clear why the artificial separation between 'prefix
bytes' and 'opcode bytes' doesn't make much sense (a little spoiler: the CB
prefix byte is the actual 'opcode', and the 'opcode byte' is just a regular
immediate value that's decoded as an opcode byte).

## Instruction Timing: M-cycles versus T-cycles

The above 'physical shape' of Z80 instructions doesn't tell us much what actually happens
during execution of an instruction (e.g. how long the instruction takes to execute,
and how the internal and externally visible state of the CPU changes during execution).

The Z80 netlist simulation is perfect for this because it allows us to inspect the internal
and observable CPU state after each clock cycle (actually: after each **half**-clock-cycle).

But first some explanation of another Z80 oddity: When reading Z80 documentation there's
a lot of talk about so called "M-cycles" and "T-cycles", often written as **M1/T2** or
**M3/T1** which confused me to no end in the beginning.

Long story short:

**M1/T2** simply means "the second clock cycle (T2) in the first machine cycle (M1)", likewise,
**M3/T1** means "the first clock cycle (T1) in the third machine cycle (M3)".

So M- and T-cycles are just a sepcial notation to identify a specific clock cycle in an instruction.

"T-cycle" means "time cycle", and it's duration is the same as a clock cycle
(which is a full "on/off" cycle of the CLK pin located in the bottom-right
corner of the chip):

![z80_clock_pin]({{ site.url }}/images/z80_clock_pin.jpg)

"M-Cycle" means "machine cycle" and simply means a related group of T-cycles. On the Z80,
basic operations like reading or writing a memory byte take more time than a single clock
cycle. But it's useful to understand the action of reading or writing a memory byte
as a single step, and that's exactly what a "machine cycle" is.

Machine cycles come in 6 main flavours:

- **Opcode Fetch** (aka M1 cycle): this is always the first (and sometimes only) machine
cycle in an instruction and takes 4 clock cycles
- **Memory Read**: read a byte from memory (3 clock cycles)
- **Memory Write**: write a byte to memory (3 clock cycles)
- **IO Read**: read a byte from an IO port (4 clock cycles)
- **IO Write**: write a byte to an IO port (4 clock cycles)
- **Internal**: an internal 'processing' machine cycle of variable length

All instruction are built from those machine cycle types, but as always on the Z80,
there's more: at the start of interrupt handling, a special machine cycle is executed.
Let's ignore that for now and come back to it later in the section about interrupts.

Since machine cycles are the basic building blocks of all instructions, it helps to understand
what exactly happens during their execution.

This is where the 'tracelog' of the Z80 netlist simulation comes in. This is a window which records
and visualizes CPU state (chip pins and register values) for each "half-clock-cycle":

![z80_tracelog]({{ site.url }}/images/z80_tracelog.jpg)

To simplify integration into blog posts like this, I added a function to dump the trace log into a text file.

An **Opcode Fetch** machine looks like this in the tracelog (with all the relevant CPU state visible):

```
OPCODE FETCH:
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │      │      │    │ 0004 │ 47 │ 0004 │ 47 │ 22 │ 03 │     M1/T1-
│ M1 │ MREQ │      │ RD │ 0004 │ 47 │ 0005 │ 47 │ 22 │ 03 │     M1/T1+
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 47 │ 22 │ 03 │     M1/T2-
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 00 │ 22 │ 03 │     M1/T2+
│    │      │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 03 │     M1/T3-
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │     M1/T3+
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │     M1/T4-
│    │      │ RFSH │    │ 2200 │ 00 │ 0005 │ 00 │ 22 │ 04 │     M1/T4+
```

Keep in mind that this shows half-clock-cycles, a 4-clock-cycle opcode fetch machine
cycle is shown as 8 half-clock-cycles in the trace log.

The **M1, MREQ, RFSH and RD** columns show the current state of the respective CPU
pins.

**AB** and **DB** are "address bus" and "data bus". **IR** is an internal register
which holds the current opcode byte. **I** and **R** are the respective CPU registers
(I is the upper byte of the interrupt vector, R is the 'refresh counter' register).

Let's go through each half-cycle:

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │      │      │    │ 0004 │ 47 │ 0004 │ 47 │ 22 │ 03 │     M1/T1-
```
The M1 pin is set to active, and the address bus has been loaded with
the current program counter PC. The data bus and instruction register
still have their values from the last instruction set (which happened
to an **LD I,A** instruction (byte sequence: ED 47)).

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │ MREQ │      │ RD │ 0004 │ 47 │ 0005 │ 47 │ 22 │ 03 │     M1/T1+
```
In the next half cycle, the MREQ and RD pins have been set in addition
to M1, which initiates a memory read from the address that's currently
on the address bus (0004). The program counter has been incremented
to the next address.

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 47 │ 22 │ 03 │     M1/T2-
```
Now the memory system has responded to the memory read request
by putting the content of address 0004 onto the data bus, which
happens to be 00 (which is a NOP instruction). The instruction
register hasn't been updated yet.

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 00 │ 22 │ 03 │     M1/T2+
```
In the next half cycle the 00 value on the data bus has been
written into the instruction register. This concludes the first
half of the opcode fetch machine cycle.

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│    │      │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 03 │     M1/T3-
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │     M1/T3+
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │     M1/T4-
│    │      │ RFSH │    │ 2200 │ 00 │ 0005 │ 00 │ 22 │ 04 │     M1/T4+
```
The remaining 4 half-cycles (2 clock cycles) are spent with
the Z80-specific 'memory refresh' while the instruction decoder
figures out what to do next (in this case: nothing, because
it's a NOP instruction). Note how the 16-bit instruction pair
made of the I and R register is put onto the address bus, and how
the R register is incremented. Also, I haven't figured out so far
why the lower 8 bits on the address bus are cleared in the very
last T-Cycle.

Let's quickly go over the remaining machine cycle types for completeness:

A **memory read** machine cycle looks like (in this case to load the byte value 22 from
address 0001 into the register L):
```
MEM READ:
┌──────┬────┬──────┬────┬──────┐
│ MREQ │ RD │ AB   │ DB │ HL   │
├──────┼────┼──────┼────┼──────┤
│      │    │ 0001 │ 21 │ 5555 │    T1-
│ MREQ │ RD │ 0001 │ 21 │ 5555 │    T1+
│ MREQ │ RD │ 0001 │ 22 │ 5555 │    T2-
│ MREQ │ RD │ 0001 │ 22 │ 5555 │    T2+
│ MREQ │ RD │ 0001 │ 22 │ 5555 │    T3-
│      │    │ 0000 │ 22 │ 5522 │    T3+
```

Here's a **memory write** machine cycle to store the value in register A (33)
into the address in register HL (1122):

```
MEM WRITE:
┌──────┬────┬──────┬────┬──────┬──────┐
│ MREQ │ WR │ AB   │ DB │ AF   │ HL   │
├──────┼────┼──────┼────┼──────┼──────┤
│      │    │ 1122 │ 77 │ 3355 │ 1122 │     T1-
│ MREQ │    │ 1122 │ 33 │ 3355 │ 1122 │     T1+
│ MREQ │    │ 1122 │ 33 │ 3355 │ 1122 │     T2-
│ MREQ │ WR │ 1122 │ 33 │ 3355 │ 1122 │     T2+
│ MREQ │ WR │ 1122 │ 33 │ 3355 │ 1122 │     T3-
│      │    │ 1122 │ 33 │ 3355 │ 1122 │     T3+
```
Note how the MREQ pin, address and data bus already contain the required values in
the second half cycle (T1 +), but the WR (write) pin is only set active in the
4th half cycle (T2 +).

The IO read and write machine cycles look similar, but are one clock cycle longer,
and setting the CPU pins is delayed by a half-clock-cycle.

```
IO READ:
┌──────┬────┬────┬──────┬────┐
│ IORQ │ RD │ WR │ AB   │ DB │
├──────┼────┼────┼──────┼────┤
│      │    │    │ 1122 │ 78 │      T1-
│      │    │    │ 1122 │ 78 │      T1+
│ IORQ │ RD │    │ 1122 │ 33 │      T2-
│ IORQ │ RD │    │ 1122 │ 33 │      T2+
│ IORQ │ RD │    │ 1122 │ 33 │      T3-
│ IORQ │ RD │    │ 1122 │ 33 │      T3+
│ IORQ │ RD │    │ 1122 │ 33 │      T4-
│      │    │    │ 1122 │ 33 │      T4+
```

```
IO WRITE:
┌──────┬────┬────┬──────┬────┐
│ IORQ │ RD │ WR │ AB   │ DB │
├──────┼────┼────┼──────┼────┤
│      │    │    │ 1122 │ 79 │      T1-
│      │    │    │ 1122 │ 21 │      T1+
│ IORQ │    │ WR │ 1122 │ 21 │      T2-
│ IORQ │    │ WR │ 1122 │ 21 │      T2+
│ IORQ │    │ WR │ 1122 │ 21 │      T3-
│ IORQ │    │ WR │ 1122 │ 21 │      T3+
│ IORQ │    │ WR │ 1122 │ 21 │      T4-
│      │    │    │ 1122 │ 21 │      T4+
```
It's interesting here that the 'pin timing' is identical between IO reads and
writes. The WR pin is activated at the same moment as the IORQ pin, while in
memory read machine cycles, the WR pin is activated two half cycles after the
MREQ pin.




