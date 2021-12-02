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

## The physical shape of Z80 instructions

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

## Instruction Timing: M-cycles and T-states

The above 'physical shape' of Z80 instructions doesn't tell us much what actually happens
during execution of an instruction (e.g. how long the instruction takes to execute,
and how the internal and externally visible state of the CPU changes during execution).

The Z80 netlist simulation is perfect for this because it allows us to inspect the internal
and observable CPU state after each clock cycle (actually: after each **half**-clock-cycle).

But first some explanation of another Z80 oddity: When reading Z80 documentation there's
a lot of talk about so called "M-cycles" and "T-states", often written as **M1/T2** or
**M3/T1** which confused me to no end in the beginning.

Long story short:

**M1/T2** simply means "the second clock cycle (T2) in the first machine cycle (M1)", likewise,
**M3/T1** means "the first clock cycle (T1) in the third machine cycle (M3)".

So M-cycles and T-states are just a special notation to identify a specific clock cycle in an instruction.

"T-state" is equivalent with a clock cycle (which is a full "on/off" cycle of the CLK pin 
located in the bottom-right corner of the chip):

![z80_clock_pin]({{ site.url }}/images/z80_clock_pin.jpg)

"M-Cycle" means "machine cycle" and simply means a related group of T-states or
clock cycles. On the Z80, basic operations like reading or writing a memory byte
take more time than a single clock cycle. But it's useful to understand the
action of reading or writing a memory byte as a single step, and that's exactly
what a "machine cycle" is.

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
│ M1 │      │      │    │ 0004 │ 47 │ 0004 │ 47 │ 22 │ 03 │  T1/0
│ M1 │ MREQ │      │ RD │ 0004 │ 47 │ 0005 │ 47 │ 22 │ 03 │  T1/1
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 47 │ 22 │ 03 │  T2/0
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 00 │ 22 │ 03 │  T2/1
│    │      │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 03 │  T3/0
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │  T3/1
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │  T4/0
│    │      │ RFSH │    │ 2200 │ 00 │ 0005 │ 00 │ 22 │ 04 │  T4/1
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
│ M1 │      │      │    │ 0004 │ 47 │ 0004 │ 47 │ 22 │ 03 │  T1/0
```
The M1 pin is set to active, and the address bus has been loaded with
the current program counter PC. The data bus and instruction register
still have their values from the last instruction set (which happened
to an **LD I,A** instruction (byte sequence: ED 47)).

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │ MREQ │      │ RD │ 0004 │ 47 │ 0005 │ 47 │ 22 │ 03 │  T1/1
```
In the next half cycle, the MREQ and RD pins have been set in addition
to M1, which initiates a memory read from the address that's currently
on the address bus (0004). The program counter has been incremented
to the next address.

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 47 │ 22 │ 03 │  T2/0
```
Now the memory system has responded to the memory read request
by putting the content of address 0004 onto the data bus, which
happens to be 00 (which is a NOP instruction). The instruction
register hasn't been updated yet.

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│ M1 │ MREQ │      │ RD │ 0004 │ 00 │ 0005 │ 00 │ 22 │ 03 │  T2/1
```
In the next half cycle the 00 value on the data bus has been
written into the instruction register. This concludes the first
half of the opcode fetch machine cycle.

```
┌────┬──────┬──────┬────┬──────┬────┬──────┬────┬────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ AB   │ DB │ PC   │ IR │ I  │ R  │
├────┼──────┼──────┼────┼──────┼────┼──────┼────┼────┼────┤
│    │      │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 03 │  T3/0
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │  T3/1
│    │ MREQ │ RFSH │    │ 2203 │ 00 │ 0005 │ 00 │ 22 │ 04 │  T4/0
│    │      │ RFSH │    │ 2200 │ 00 │ 0005 │ 00 │ 22 │ 04 │  T4/1
```
The remaining 4 half-cycles (2 clock cycles) are spent with
the Z80-specific 'memory refresh' while the instruction decoder
figures out what to do next (in this case: nothing, because
it's a NOP instruction). Note how the 16-bit instruction pair
made of the I and R register is put onto the address bus, and how
the R register is incremented. Also, I haven't figured out so far
why the lower 8 bits on the address bus are cleared in the very
last half-clock-cycle.

Let's quickly go over the remaining machine cycle types for completeness:

A **memory read** machine cycle looks like (in this case to load the byte value 22 from
address 0001 into the register L):
```
MEM READ:
┌──────┬────┬──────┬────┬──────┐
│ MREQ │ RD │ AB   │ DB │ HL   │
├──────┼────┼──────┼────┼──────┤
│      │    │ 0001 │ 21 │ 5555 │  T1/0
│ MREQ │ RD │ 0001 │ 21 │ 5555 │  T1/1
│ MREQ │ RD │ 0001 │ 22 │ 5555 │  T2/0
│ MREQ │ RD │ 0001 │ 22 │ 5555 │  T2/1
│ MREQ │ RD │ 0001 │ 22 │ 5555 │  T3/0
│      │    │ 0000 │ 22 │ 5522 │  T3/1
```

Here's a **memory write** machine cycle to store the value in register A (33)
into the address in register HL (1122):

```
MEM WRITE:
┌──────┬────┬──────┬────┬──────┬──────┐
│ MREQ │ WR │ AB   │ DB │ AF   │ HL   │
├──────┼────┼──────┼────┼──────┼──────┤
│      │    │ 1122 │ 77 │ 3355 │ 1122 │  T1/0
│ MREQ │    │ 1122 │ 33 │ 3355 │ 1122 │  T1/1
│ MREQ │    │ 1122 │ 33 │ 3355 │ 1122 │  T2/0
│ MREQ │ WR │ 1122 │ 33 │ 3355 │ 1122 │  T2/1
│ MREQ │ WR │ 1122 │ 33 │ 3355 │ 1122 │  T3/0
│      │    │ 1122 │ 33 │ 3355 │ 1122 │  T3/1
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
│      │    │    │ 1122 │ 78 │  T1/0
│      │    │    │ 1122 │ 78 │  T1/1
│ IORQ │ RD │    │ 1122 │ 33 │  T2/0
│ IORQ │ RD │    │ 1122 │ 33 │  T2/1
│ IORQ │ RD │    │ 1122 │ 33 │  T3/0
│ IORQ │ RD │    │ 1122 │ 33 │  T3/1
│ IORQ │ RD │    │ 1122 │ 33 │  T4/0
│      │    │    │ 1122 │ 33 │  T4/1
```

```
IO WRITE:
┌──────┬────┬────┬──────┬────┐
│ IORQ │ RD │ WR │ AB   │ DB │
├──────┼────┼────┼──────┼────┤
│      │    │    │ 1122 │ 79 │  T1/0
│      │    │    │ 1122 │ 21 │  T1/1
│ IORQ │    │ WR │ 1122 │ 21 │  T2/0
│ IORQ │    │ WR │ 1122 │ 21 │  T2/1
│ IORQ │    │ WR │ 1122 │ 21 │  T3/0
│ IORQ │    │ WR │ 1122 │ 21 │  T3/1
│ IORQ │    │ WR │ 1122 │ 21 │  T4/0
│      │    │    │ 1122 │ 21 │  T4/1
```
It's interesting here that the 'pin timing' is identical between IO reads and
writes. The WR pin is activated at the same moment as the IORQ pin, while in
memory read machine cycles, the WR pin is activated two half cycles after the
MREQ pin.

## Instruction timing: wait states

All machine cycles that access memory or IO check the WAIT input pin at exactly
one clock cycle. If the WAIT pin is active, the execution 'freezes' until
the WAIT pin goes inactive. The original intent was to give slow memory and
IO devices time to react, but some computer systems also use wait states in more
creative ways to memory access between CPU and video hardware (for instance on
the Amstrad CPC).

The exact T-state where the wait pin is samples depends on the machine cycle
type.

In the opcode fetch machine cycle, the wait pin is sampled in the second half-cycle
of M1/T2. If the WAIT pin isn't active in this exact half cycle, the CPU will not go
into wait state mode, otherwise the CPU will insert extra 'wait cycles' until the
WAIT pin goes inactive. 

For instance if the WAIT pin is only active in the second half cycle of T2, the opcode
fetch machine cycle will be stretched from 4 to 5 clock cycles:

```
OPCODE FETCH:
┌────┬──────┬──────┬────┬────┬──────┬──────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ WAIT │ AB   │ DB │
├────┼──────┼──────┼────┼────┼──────┼──────┼────┤
│ M1 │      │      │    │    │      │ 0000 │ 00 │  T1/0
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 00 │  T1/1
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T2/0
│ M1 │ MREQ │      │ RD │    │ WAIT │ 0000 │ 31 │  T2/1    <== WAIT pin sampled here
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T3/0    <== one extra clock cycle inserted
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T3/1    
│    │      │ RFSH │    │    │      │ 0000 │ 31 │  T4/0    <== regular execution continues here
│    │ MREQ │ RFSH │    │    │      │ 0000 │ 31 │  T4/1
│    │ MREQ │ RFSH │    │    │      │ 0000 │ 31 │  T5/0
│    │      │ RFSH │    │    │      │ 0000 │ 31 │  T5/1
```

If the wait pin goes inactive in the first half cycle, the CPU will leave the wait state mode
at the end of the clock cycle:

```
OPCODE FETCH:
┌────┬──────┬──────┬────┬────┬──────┬──────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ WAIT │ AB   │ DB │
├────┼──────┼──────┼────┼────┼──────┼──────┼────┤
│ M1 │      │      │    │    │      │ 0000 │ 00 │  T1/0
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 00 │  T1/1
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T2/0
│ M1 │ MREQ │      │ RD │    │ WAIT │ 0000 │ 31 │  T2/1    <== WAIT pin sampled here
│ M1 │ MREQ │      │ RD │    │ WAIT │ 0000 │ 31 │  T3/0    <== WAIT pin active for 2 half cycles
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T3/1    <== extra clock cycle completes
│    │      │ RFSH │    │    │      │ 0000 │ 31 │  T4/0    <== regular execution continues here
│    │ MREQ │ RFSH │    │    │      │ 0000 │ 31 │  T4/1
│    │ MREQ │ RFSH │    │    │      │ 0000 │ 31 │  T5/0
│    │      │ RFSH │    │    │      │ 0000 │ 31 │  T5/1
```

Setting the WAIT pin until the second half cycle causes one more clock cycle to be inserted:

```
OPCODE FETCH:
┌────┬──────┬──────┬────┬────┬──────┬──────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ WAIT │ AB   │ DB │
├────┼──────┼──────┼────┼────┼──────┼──────┼────┤
│ M1 │      │      │    │    │      │ 0000 │ 00 │  T1/0
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 00 │  T1/1
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T2/0
│ M1 │ MREQ │      │ RD │    │ WAIT │ 0000 │ 31 │  T2/1    <== WAIT pin sampled here
│ M1 │ MREQ │      │ RD │    │ WAIT │ 0000 │ 31 │  T3/0    <== WAIT pin active for 3 half cycles
│ M1 │ MREQ │      │ RD │    │ WAIT │ 0000 │ 31 │  T3/1    <== first inserted clock cycle completes
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T4/0    <== a second wait clock cycle is inserted    
│ M1 │ MREQ │      │ RD │    │      │ 0000 │ 31 │  T4/1
│    │      │ RFSH │    │    │      │ 0000 │ 31 │  T5/0    <== regular execution continues here
│    │ MREQ │ RFSH │    │    │      │ 0000 │ 31 │  T5/1
│    │ MREQ │ RFSH │    │    │      │ 0000 │ 31 │  T6/0
│    │      │ RFSH │    │    │      │ 0000 │ 31 │  T6/1
```

In memory read machine cycles, the WAIT pin is sampled at the second
half cycle of T2 (same as in an opcode fetch).
```
MEM READ:
┌──────┬────┬────┬──────┬──────┬────┐
│ MREQ │ RD │ WR │ WAIT │ AB   │ DB │
├──────┼────┼────┼──────┼──────┼────┤
│      │    │    │      │ 0001 │ 31 │  T1/0
│ MREQ │ RD │    │      │ 0001 │ 31 │  T1/1
│ MREQ │ RD │    │      │ 0001 │ 30 │  T2/0
│ MREQ │ RD │    │ WAIT │ 0001 │ 30 │  T2/1    <== WAIT pin sampled here
│ MREQ │ RD │    │      │ 0001 │ 30 │  T3/0    <== extra clock cycle
│ MREQ │ RD │    │      │ 0001 │ 30 │  T3/1
│ MREQ │ RD │    │      │ 0001 │ 30 │  T4/0    <== regular execution continues
│      │    │    │      │ 0000 │ 30 │  T4/1
```

In memory write machine cycles, the WAIT pin is also sampled at the
second half cycle of T2:

```
MEM WRITE:
┌──────┬────┬────┬──────┬──────┬────┐
│ MREQ │ RD │ WR │ WAIT │ AB   │ DB │
├──────┼────┼────┼──────┼──────┼────┤
│      │    │    │      │ 1234 │ 77 │  T1/0    
│ MREQ │    │    │      │ 1234 │ 11 │  T1/1
│ MREQ │    │    │      │ 1234 │ 11 │  T2/0
│ MREQ │    │ WR │ WAIT │ 1234 │ 11 │  T2/1    <== WAIT pin sampled here
│ MREQ │    │ WR │      │ 1234 │ 11 │  T3/0    <== extra clock cycle
│ MREQ │    │ WR │      │ 1234 │ 11 │  T3/1
│ MREQ │    │ WR │      │ 1234 │ 11 │  T4/0    <== regular execution continues
│      │    │    │      │ 1234 │ 11 │  T4/1
```

In IO read and write machine cycles, the WAIT pin is samples one full clock 
cycle later, at the second half cycle of T3:
```
IO READ:
┌──────┬────┬────┬──────┬──────┬────┐
│ IORQ │ RD │ WR │ WAIT │ AB   │ DB │
├──────┼────┼────┼──────┼──────┼────┤
│      │    │    │      │ 1234 │ 78 │  T1/0
│      │    │    │      │ 1234 │ 78 │  T1/1
│ IORQ │ RD │    │      │ 1234 │ FF │  T2/0
│ IORQ │ RD │    │      │ 1234 │ FF │  T2/1
│ IORQ │ RD │    │      │ 1234 │ FF │  T3/0
│ IORQ │ RD │    │ WAIT │ 1234 │ FF │  T3/1    <== WAIT pin sampled here
│ IORQ │ RD │    │      │ 1234 │ FF │  T4/0    <== extra clock cycle
│ IORQ │ RD │    │      │ 1234 │ FF │  T4/1
│ IORQ │ RD │    │      │ 1234 │ FF │  T5/0    <== regular execution continues
│      │    │    │      │ 1234 │ FF │  T5/1
```

```
IO WRITE:
┌──────┬────┬────┬──────┬──────┬────┐
│ IORQ │ RD │ WR │ WAIT │ AB   │ DB │
├──────┼────┼────┼──────┼──────┼────┤
│      │    │    │      │ 1234 │ 79 │  T1/0
│      │    │    │      │ 1234 │ 11 │  T1/1
│ IORQ │    │ WR │      │ 1234 │ 11 │  T2/0
│ IORQ │    │ WR │      │ 1234 │ 11 │  T2/1
│ IORQ │    │ WR │      │ 1234 │ 11 │  T3/0
│ IORQ │    │ WR │ WAIT │ 1234 │ 11 │  T3/1    <== WAIT pin sampled here
│ IORQ │    │ WR │      │ 1234 │ 11 │  T4/0    <== extra clock cycle
│ IORQ │    │ WR │      │ 1234 │ 11 │  T4/1
│ IORQ │    │ WR │      │ 1234 │ 11 │  T5/0    <== regular execution continues
│      │    │    │      │ 1234 │ 11 │  T5/1
```

## Instruction Timing: Extra Cycles

With the knowledge that machine cycles are the basic building blocks of instructions, and
the length of those machine cycles we should be able to 'prefict' the number
of clock cycles an instruction takes.

For instance the instruction *LD HL,nnnn* (load 16-bit immediate value into register pair *HL*)
consists of the following machine cycles

1. opcode fetch (4 clock cycles) to read the opcode byte
2. a memory read (3 clock cycles) to readt the next byte into L
3. and another memory read (3 clock cycles) to read the next byte into H

Together: **4 + 3 + 3 = 10 clock cycles**, which is totally correct.

The instruction **PUSH HL** (push content of HL register on the stack) should be the same,
except that memory reads are replaced with memory writes:

1. opcode fetch (4 clock cycles)
2. a memory write to write H to the stack
3. another memory write to write L to the stack

This *should* take 10 clock cycles too, but **PUSH HL** actually takes 11 clock cycles
to execute:

```
PUSH HL:
┌────┬──────┬──────┬────┬────┬──────┬────┬──────┬──────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AB   │ DB │ HL   │ SP   │
├────┼──────┼──────┼────┼────┼──────┼────┼──────┼──────┤
│ M1 │      │      │    │    │ 0006 │ 12 │ 1234 │ 0100 │  M1/T1/0 (opcode fetch)
│ M1 │ MREQ │      │ RD │    │ 0006 │ 12 │ 1234 │ 0100 │  M1/T1/1
│ M1 │ MREQ │      │ RD │    │ 0006 │ E5 │ 1234 │ 0100 │  M1/T2/0
│ M1 │ MREQ │      │ RD │    │ 0006 │ E5 │ 1234 │ 0100 │  M1/T2/1
│    │      │ RFSH │    │    │ 0002 │ E5 │ 1234 │ 0100 │  M1/T3/0
│    │ MREQ │ RFSH │    │    │ 0002 │ E5 │ 1234 │ 0100 │  M1/T3/1
│    │ MREQ │ RFSH │    │    │ 0002 │ E5 │ 1234 │ 0100 │  M1/T4/0
│    │      │ RFSH │    │    │ 0002 │ E5 │ 1234 │ 0100 │  M1/T4/1
│    │      │      │    │    │ 0002 │ E5 │ 1234 │ 0100 │  <== WTF???
│    │      │      │    │    │ 0000 │ E5 │ 1234 │ 00FF │  <== WTF???
│    │      │      │    │    │ 00FF │ E5 │ 1234 │ 00FF │  M2/T1/0 (memory write)
│    │ MREQ │      │    │    │ 00FF │ 12 │ 1234 │ 00FF │  M2/T1/1
│    │ MREQ │      │    │    │ 00FF │ 12 │ 1234 │ 00FF │  M2/T2/0
│    │ MREQ │      │    │ WR │ 00FF │ 12 │ 1234 │ 00FE │  M2/T2/1
│    │ MREQ │      │    │ WR │ 00FF │ 12 │ 1234 │ 00FE │  M2/T3/0
│    │      │      │    │    │ 00FE │ 12 │ 1234 │ 00FE │  M2/T3/1
│    │      │      │    │    │ 00FE │ E5 │ 1234 │ 00FE │  M3/T1/0 (memory write)
│    │ MREQ │      │    │    │ 00FE │ 34 │ 1234 │ 00FE │  M3/T1/1
│    │ MREQ │      │    │    │ 00FE │ 34 │ 1234 │ 00FE │  M3/T2/0
│    │ MREQ │      │    │ WR │ 00FE │ 34 │ 1234 │ 00FE │  M3/T2/1
│    │ MREQ │      │    │ WR │ 00FE │ 34 │ 1234 │ 00FE │  M3/T3/0
│    │      │      │    │    │ 00FE │ 34 │ 1234 │ 00FE │  M3/T3/1
```

There's an additional clock cycle squeezed inbetween the opcode fetch and
first memory read machine cycle which is used to 'pre-decrement'
the *SP* register before the memory write machine cycles can happen.

It's little irregularities like this which complicate writing a Z80 emulator.
In a cycle correct emulator it is not only important that instructions
take the correct number of clock cycles to execute, but also that
memory and IO reads/writes happen at the correct clock cycle.

## Overlapped Execution

In some instructions, execution 'leaks' into the opcode fetch machine cycle
of the next instruction.

For instance when inspecting the instruction 'XOR A' (which clears the A register
and sets flags accordingly the instruction doesn't seem to have any effect:

```
XOR A:
┌────┬──────┬──────┬────┬────┬──────┬──────────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AF   │ Flags    │
├────┼──────┼──────┼────┼────┼──────┼──────────┤
│ M1 │      │      │    │    │ FFAC │ SzYhXVnc │  T1/0
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │  T1/1
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │  T2/0
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │  T2/1
│    │      │ RFSH │    │    │ FFAC │ SzYhXVnc │  T3/0
│    │ MREQ │ RFSH │    │    │ FFAC │ SzYhXVnc │  T3/1
│    │ MREQ │ RFSH │    │    │ FFAC │ SzYhXVnc │  T4/0
│    │      │ RFSH │    │    │ FFAC │ SzYhXVnc │  T4/1
                               ^^     ^^^^^^^^
```

*XOR A* takes 4 clock cycles, yet at the end of the instruction A isn't zero,
and flag bits haven't been updated either. Here's the same diagram including
the *NOP* instruction that follows:

```
XOR A + NOP:
┌────┬──────┬──────┬────┬────┬──────┬──────────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AF   │ Flags    │
├────┼──────┼──────┼────┼────┼──────┼──────────┤
│ M1 │      │      │    │    │ FFAC │ SzYhXVnc │ T1/0 <== XOR A start
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │ T1/1
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │ T2/0
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │ T2/1
│    │      │ RFSH │    │    │ FFAC │ SzYhXVnc │ T3/0
│    │ MREQ │ RFSH │    │    │ FFAC │ SzYhXVnc │ T3/1
│    │ MREQ │ RFSH │    │    │ FFAC │ SzYhXVnc │ T4/0
│    │      │ RFSH │    │    │ FFAC │ SzYhXVnc │ T4/1
├────┼──────┼──────┼────┼────┼──────┼──────────┤
│ M1 │      │      │    │    │ FFAC │ SzYhXVnc │ T1/0 <== NOP starts here
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │ T1/1
│ M1 │ MREQ │      │ RD │    │ FFAC │ SzYhXVnc │ T2/0
│ M1 │ MREQ │      │ RD │    │ 00AC │ SzYhXVnc │ T2/1 <== A updated here
│    │      │ RFSH │    │    │ 00AC │ SzYhXVnc │ T3/0
│    │ MREQ │ RFSH │    │    │ 0044 │ sZyhxVnc │ T3/1 <== flags updated here 
│    │ MREQ │ RFSH │    │    │ 0044 │ sZyhxVnc │ T4/0
│    │      │ RFSH │    │    │ 0044 │ sZyhxVnc │ T4/1
```

The results of the *XOR A* instruction only become available
at the end of the second and third clock cycles of the following instruction.

Thankfully this overlapped instruction execution is hardly relevant for CPU
emulators, because it only affects the internal state of the CPU, not any state
that's observable from the outside. The result of an instruction is relevant at
the earliest *after* the opcode fetch machine cycle of the next instruction has
finished.

## The 3 Instruction Subsets

The Z80 instruction set is really 3 separate subsets each occupying
256 opcodes. There's the main instruction set which mostly overlaps
with the Intel 8080 instruction set and two additional sets of instructions
selected with the *ED* and *CB* prefix opcodes.

The main and CB subsets each occupy the full range of 256 instructions,
while the ED subset is mostly empty and only implements 59 instructions.

I'm not counting the *DD* and *FD* prefix instruction ranges as separate
subsets because they only slightly modify the behaviour of the main
instructions.

This means there are 571 unique instructions in the Z80 instruction set
(I'm counting the RETI and RETN instruction as one because they have 
identical behaviour).

## The 2-3-3 Opcode Bit Pattern

Opcode bytes can be split into three bit groups to reveal a hidden
'octal structure' of a 256 instruction subset:

```
  7 6   5 4 3   2 1 0
| x x | y y y | z z z | 
```

The two topmost bits (xx) split the instruction space into 4 quadrants,
and the remaining 6 bits are divided into two 3-bit groups (yyy) and (zzz)
which are used by the instruction decoder to select a specific group
of instructions or as an 'register index'.

Let's look at each instruction subset and quadrant one by one:

## Main Quadrant 1 (xx = 01):

I'm starting with Main Quadrant 1 (not 0) because unlike 0 it is has such
a simple structure. In an Z80 emulator this is usually the first quadrant
I'm implementing.

The 64 instructions with the bit pattern **|01|yyy|zzz|** implement
8-bit move instructions where **yyy** defines the target and **zzz**
the source. As a table, the main quadrant 1 looks like this:

<style>
.z80t { border:1px solid black;border-collapse:collapse;padding:5px; }
.z80h { border:1px solid black;border-collapse:collapse;padding:5px;color:black;background-color:Gainsboro }
.z80c0 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:PaleGreen; }
.z80c1 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:LightPink; }
</style>
<table class="z80t">
<tr class="z80t"><th class="z80h">x=01</th><th class="z80h">z=000</th><th class="z80h">z=001</th><th class="z80h">z=010</th><th class="z80h">z=011</th><th class="z80h">z=100</th><th class="z80h">z=101</th><th class="z80h">z=110</th><th class="z80h">z=111</th></tr><tr class="z80t"><th class="z80h">y=000</th><td class="z80c0">LD B,B</td><td class="z80c0">LD B,C</td><td class="z80c0">LD B,D</td><td class="z80c0">LD B,E</td><td class="z80c0">LD B,H</td><td class="z80c0">LD B,L</td><td class="z80c0">LD B,(HL)</td><td class="z80c0">LD B,A</td></tr><tr class="z80t"><th class="z80h">y=001</th><td class="z80c0">LD C,B</td><td class="z80c0">LD C,C</td><td class="z80c0">LD C,D</td><td class="z80c0">LD C,E</td><td class="z80c0">LD C,H</td><td class="z80c0">LD C,L</td><td class="z80c0">LD C,(HL)</td><td class="z80c0">LD C,A</td></tr><tr class="z80t"><th class="z80h">y=010</th><td class="z80c0">LD D,B</td><td class="z80c0">LD D,C</td><td class="z80c0">LD D,D</td><td class="z80c0">LD D,E</td><td class="z80c0">LD D,H</td><td class="z80c0">LD D,L</td><td class="z80c0">LD D,(HL)</td><td class="z80c0">LD D,A</td></tr><tr class="z80t"><th class="z80h">y=011</th><td class="z80c0">LD E,B</td><td class="z80c0">LD E,C</td><td class="z80c0">LD E,D</td><td class="z80c0">LD E,E</td><td class="z80c0">LD E,H</td><td class="z80c0">LD E,L</td><td class="z80c0">LD E,(HL)</td><td class="z80c0">LD E,A</td></tr><tr class="z80t"><th class="z80h">y=100</th><td class="z80c0">LD H,B</td><td class="z80c0">LD H,C</td><td class="z80c0">LD H,D</td><td class="z80c0">LD H,E</td><td class="z80c0">LD H,H</td><td class="z80c0">LD H,L</td><td class="z80c0">LD H,(HL)</td><td class="z80c0">LD H,A</td></tr><tr class="z80t"><th class="z80h">y=101</th><td class="z80c0">LD L,B</td><td class="z80c0">LD L,C</td><td class="z80c0">LD L,D</td><td class="z80c0">LD L,E</td><td class="z80c0">LD L,H</td><td class="z80c0">LD L,L</td><td class="z80c0">LD L,(HL)</td><td class="z80c0">LD L,A</td></tr><tr class="z80t"><th class="z80h">y=110</th><td class="z80c0">LD (HL),B</td><td class="z80c0">LD (HL),C</td><td class="z80c0">LD (HL),D</td><td class="z80c0">LD (HL),E</td><td class="z80c0">LD (HL),H</td><td class="z80c0">LD (HL),L</td><td class="z80c0">HALT</td><td class="z80c0">LD (HL),A</td></tr><tr class="z80t"><th class="z80h">y=111</th><td class="z80c0">LD A,B</td><td class="z80c0">LD A,C</td><td class="z80c0">LD A,D</td><td class="z80c0">LD A,E</td><td class="z80c0">LD A,H</td><td class="z80c0">LD A,L</td><td class="z80c0">LD A,(HL)</td><td class="z80c0">LD A,A</td></tr>
</table><br>

*y* and *z* are registers indices as binary numbers:

```
000 = 0 => B
001 = 1 => C
010 = 2 => D
011 = 3 => E
100 = 4 => H
101 = 5 => L
110 = 6 => (HL)
111 = 7 => A
```

The 'register index' 6 is a bit special. Normally this index should be used for
the F (flags) register, which isn't directly accessible. Instead this index is
used as special case to load or store the 8-bit values addressed by the register
pair HL.

And another oddity is the HALT instruction at bit pattern **|01|110|110|** (==
76 hex). Following the 'table logic' this instruction slot *should* be occupied by
an **LD (HL),(HL)** instruction which doesn't make a lot of sense, so instead
this slot was reused for the HALT instruction.

I have choosen the green background for instructions that have no 'timing
surprises' (like the **PUSH HL** instruction discussed above).  The duration
of 'green' instructions is simply the sum of their default machine cycle
lengths.  All instructions in the Main Quadrant 1 take 4 clock cycles (for the
opcode fetch), except the instructions involving **(HL)** which take an additional
memory read or write machine cycle, resulting in 7 clock cycles.

## Main Quadrant 2 (xx = 10)

This is the second 'beautiful' quadrant in the main instruction set, this
is where the basic 8-bit ALU instructions live:

<style>
.z80t { border:1px solid black;border-collapse:collapse;padding:5px; }
.z80h { border:1px solid black;border-collapse:collapse;padding:5px;color:black;background-color:Gainsboro }
.z80c0 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:PaleGreen; }
.z80c1 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:LightPink; }
</style>
<table class="z80t">
<tr class="z80t"><th class="z80h">x=10</th><th class="z80h">z=000</th><th class="z80h">z=001</th><th class="z80h">z=010</th><th class="z80h">z=011</th><th class="z80h">z=100</th><th class="z80h">z=101</th><th class="z80h">z=110</th><th class="z80h">z=111</th></tr><tr class="z80t"><th class="z80h">y=000</th><td class="z80c0">ADD B</td><td class="z80c0">ADD C</td><td class="z80c0">ADD D</td><td class="z80c0">ADD E</td><td class="z80c0">ADD H</td><td class="z80c0">ADD L</td><td class="z80c0">ADD (HL)</td><td class="z80c0">ADD A</td></tr><tr class="z80t"><th class="z80h">y=001</th><td class="z80c0">ADC B</td><td class="z80c0">ADC C</td><td class="z80c0">ADC D</td><td class="z80c0">ADC E</td><td class="z80c0">ADC H</td><td class="z80c0">ADC L</td><td class="z80c0">ADC (HL)</td><td class="z80c0">ADC A</td></tr><tr class="z80t"><th class="z80h">y=010</th><td class="z80c0">SUB B</td><td class="z80c0">SUB C</td><td class="z80c0">SUB D</td><td class="z80c0">SUB E</td><td class="z80c0">SUB H</td><td class="z80c0">SUB L</td><td class="z80c0">SUB (HL)</td><td class="z80c0">SUB A</td></tr><tr class="z80t"><th class="z80h">y=011</th><td class="z80c0">SBC B</td><td class="z80c0">SBC C</td><td class="z80c0">SBC D</td><td class="z80c0">SBC E</td><td class="z80c0">SBC H</td><td class="z80c0">SBC L</td><td class="z80c0">SBC (HL)</td><td class="z80c0">SBC A</td></tr><tr class="z80t"><th class="z80h">y=100</th><td class="z80c0">AND B</td><td class="z80c0">AND C</td><td class="z80c0">AND D</td><td class="z80c0">AND E</td><td class="z80c0">AND H</td><td class="z80c0">AND L</td><td class="z80c0">AND (HL)</td><td class="z80c0">AND A</td></tr><tr class="z80t"><th class="z80h">y=101</th><td class="z80c0">XOR B</td><td class="z80c0">XOR C</td><td class="z80c0">XOR D</td><td class="z80c0">XOR E</td><td class="z80c0">XOR H</td><td class="z80c0">XOR L</td><td class="z80c0">XOR (HL)</td><td class="z80c0">XOR A</td></tr><tr class="z80t"><th class="z80h">y=110</th><td class="z80c0">OR B</td><td class="z80c0">OR C</td><td class="z80c0">OR D</td><td class="z80c0">OR E</td><td class="z80c0">OR H</td><td class="z80c0">OR L</td><td class="z80c0">OR (HL)</td><td class="z80c0">OR A</td></tr><tr class="z80t"><th class="z80h">y=111</th><td class="z80c0">CP B</td><td class="z80c0">CP C</td><td class="z80c0">CP D</td><td class="z80c0">CP E</td><td class="z80c0">CP H</td><td class="z80c0">CP L</td><td class="z80c0">CP (HL)</td><td class="z80c0">CP A</td></tr>
</table><br>

Again, no timing surprises in this quadrant. The *z* bit group selects the
source register or **(HL)**, and the *y* bit group the ALU operation:

```
000 => 0 => ADD
001 => 1 => ADC (add with carry)
010 => 2 => SUB
011 => 3 => SBC (sub with carry)
100 => 4 => AND
101 => 5 => XOR
110 => 6 => OR
111 => 7 => CP (compare - like SUB, but discard result)
```

This table also demonstrates nicely why all ALU operations implicitely use the
register **A** to store the result.  There's simply no bits left in the 8-bit
opcode to select a destination register.

## Main Quadrant 0 (xx = 00)

This is the first of the two 'messy' quadrants in the main set:

<style>
.z80t { border:1px solid black;border-collapse:collapse;padding:5px; }
.z80h { border:1px solid black;border-collapse:collapse;padding:5px;color:black;background-color:Gainsboro }
.z80c0 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:PaleGreen; }
.z80c1 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:LightPink; }
</style>
<table class="z80t">
<tr class="z80t"><th class="z80h">x=00</th><th class="z80h">z=000</th><th class="z80h">z=001</th><th class="z80h">z=010</th><th class="z80h">z=011</th><th class="z80h">z=100</th><th class="z80h">z=101</th><th class="z80h">z=110</th><th class="z80h">z=111</th></tr><tr class="z80t"><th class="z80h">y=000</th><td class="z80c0">NOP</td><td class="z80c0">LD BC,nn</td><td class="z80c0">LD (BC),A</td><td class="z80c1">INC BC</td><td class="z80c0">INC B</td><td class="z80c0">DEC B</td><td class="z80c0">LD B,n</td><td class="z80c0">RLCA</td></tr><tr class="z80t"><th class="z80h">y=001</th><td class="z80c0">EX AF,AF'</td><td class="z80c1">ADD HL,BC</td><td class="z80c0">LD A,(BC)</td><td class="z80c1">DEC BC</td><td class="z80c0">INC C</td><td class="z80c0">DEC C</td><td class="z80c0">LD C,n</td><td class="z80c0">RRCA</td></tr><tr class="z80t"><th class="z80h">y=010</th><td class="z80c1">DJNZ d</td><td class="z80c0">LD DE,nn</td><td class="z80c0">LD (DE),A</td><td class="z80c1">INC DE</td><td class="z80c0">INC D</td><td class="z80c0">DEC D</td><td class="z80c0">LD D,n</td><td class="z80c0">RLA</td></tr><tr class="z80t"><th class="z80h">y=011</th><td class="z80c1">JR d</td><td class="z80c1">ADD HL,DE</td><td class="z80c0">LD A,(DE)</td><td class="z80c1">DEC DE</td><td class="z80c0">INC E</td><td class="z80c0">DEC E</td><td class="z80c0">LD E,n</td><td class="z80c0">RRA</td></tr><tr class="z80t"><th class="z80h">y=100</th><td class="z80c1">JR NZ,d</td><td class="z80c0">LD HL,nn</td><td class="z80c0">LD (nn),HL</td><td class="z80c1">INC HL</td><td class="z80c0">INC H</td><td class="z80c0">DEC H</td><td class="z80c0">LD H,n</td><td class="z80c0">DAA</td></tr><tr class="z80t"><th class="z80h">y=101</th><td class="z80c1">JR Z,d</td><td class="z80c1">ADD HL,HL</td><td class="z80c0">LD HL,(nn)</td><td class="z80c1">DEC HL</td><td class="z80c0">INC L</td><td class="z80c0">DEC L</td><td class="z80c0">LD L,n</td><td class="z80c0">CPL</td></tr><tr class="z80t"><th class="z80h">y=110</th><td class="z80c1">JR NC,d</td><td class="z80c0">LD SP,nn</td><td class="z80c0">LD (nn),A</td><td class="z80c1">INC SP</td><td class="z80c1">INC (HL)</td><td class="z80c1">DEC (HL)</td><td class="z80c0">LD (HL),n</td><td class="z80c0">SCF</td></tr><tr class="z80t"><th class="z80h">y=111</th><td class="z80c1">JR C,d</td><td class="z80c1">ADD HL,SP</td><td class="z80c0">LD A,(nn)</td><td class="z80c1">DEC SP</td><td class="z80c0">INC A</td><td class="z80c0">DEC A</td><td class="z80c0">LD A,n</td><td class="z80c0">CCF</td></tr>
</table><br>

The red background color means that those instructions have squeeze some extra
clock cycles inbetween the regular memory access machine cycles and need to be
handled with special care in cycle-correct emulators. 

The **INC (HL)** and **DEC (HL)** instructions stick out, those are read-modify-write
instructions. Let's see why they have a red background:

```
INC (HL):
┌────┬──────┬──────┬────┬────┬──────┬────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AB   │ DB │
├────┼──────┼──────┼────┼────┼──────┼────┤
│ M1 │      │      │    │    │ 0003 │ 12 │ M1/T1/0 <== opcode fetch
│ M1 │ MREQ │      │ RD │    │ 0003 │ 12 │ M1/T1/1
│ M1 │ MREQ │      │ RD │    │ 0003 │ 34 │ M1/T2/0
│ M1 │ MREQ │      │ RD │    │ 0000 │ 34 │ M1/T2/1
│    │      │ RFSH │    │    │ 0001 │ 34 │ M1/T3/0
│    │ MREQ │ RFSH │    │    │ 0001 │ 34 │ M1/T3/1
│    │ MREQ │ RFSH │    │    │ 0001 │ 34 │ M1/T4/0
│    │      │ RFSH │    │    │ 0000 │ 34 │ M1/T4/1
│    │      │      │    │    │ 1234 │ 34 │ M2/T1/0 <== memory read
│    │ MREQ │      │ RD │    │ 1234 │ 34 │ M2/T1/1
│    │ MREQ │      │ RD │    │ 1234 │ 00 │ M2/T2/0
│    │ MREQ │      │ RD │    │ 1234 │ 00 │ M2/T2/1
│    │ MREQ │      │ RD │    │ 1234 │ 00 │ M2/T3/0
│    │      │      │    │    │ 1234 │ 00 │ M2/T3/1
│    │      │      │    │    │ 1234 │ 00 │         <== ??? 
│    │      │      │    │    │ 1234 │ 00 │         <== ???
│    │      │      │    │    │ 1234 │ 00 │ M3/T1/0 <== memory write
│    │ MREQ │      │    │    │ 1234 │ 01 │ M3/T1/1
│    │ MREQ │      │    │    │ 1234 │ 01 │ M3/T2/0
│    │ MREQ │      │    │ WR │ 1234 │ 01 │ M3/T2/1
│    │ MREQ │      │    │ WR │ 1234 │ 01 │ M3/T3/0
│    │      │      │    │    │ 1234 │ 01 │ M3/T3/1
```

As expected, there's an opcode fetch, memory read and memory write machine cycle.
An extra clock cycle has been squeezed between the read and write machine cycle,
no doubt to increment the byte that's been loaded from memory before it is 
written back.

The 16-bit **INC/DEC** column adds two additional clock cycles after the opcode fetch
machine cycle to perform the 16-bit math:

```
INC BC:
┌────┬──────┬──────┬────┬────┬──────┬────┬──────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AB   │ DB │ BC   │
├────┼──────┼──────┼────┼────┼──────┼────┼──────┤
│ M1 │      │      │    │    │ 0003 │ FF │ FFFF │ M1/T1/0
│ M1 │ MREQ │      │ RD │    │ 0003 │ FF │ FFFF │ M1/T1/1
│ M1 │ MREQ │      │ RD │    │ 0003 │ 03 │ FFFF │ M1/T2/0
│ M1 │ MREQ │      │ RD │    │ 0000 │ 03 │ FFFF │ M1/T2/1
│    │      │ RFSH │    │    │ 0001 │ 03 │ FFFF │ M1/T3/0
│    │ MREQ │ RFSH │    │    │ 0001 │ 03 │ FFFF │ M1/T3/1
│    │ MREQ │ RFSH │    │    │ 0001 │ 03 │ FFFF │ M1/T4/0
│    │      │ RFSH │    │    │ 0001 │ 03 │ FFFF │ M1/T4/1
│    │      │      │    │    │ 0001 │ 03 │ FFFF │        <== ???
│    │      │      │    │    │ 0001 │ 03 │ 0000 │        <== ???
│    │      │      │    │    │ 0001 │ 03 │ 0000 │        <== ???
│    │      │      │    │    │ 0000 │ 03 │ 0000 │        <== ???
```
It's interesting that the result is already available at the
end of the first extra clock cycle. No idea why there's 
a second 'wasted' clock cycle, especially since the 
16-bit INC/DEC instructions don't update the flag bits. 

The 16-bit **ADD** instructions add 7 extra clock cycles after
the opcode fetch machine cycle:

```
ADD HL,DE
┌────┬──────┬──────┬────┬────┬──────┬────┬──────┬──────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AB   │ DB │ DE   │ HL   │
├────┼──────┼──────┼────┼────┼──────┼────┼──────┼──────┤
│ M1 │      │      │    │    │ 0006 │ 22 │ 2222 │ 1111 │ M1/T1/0
│ M1 │ MREQ │      │ RD │    │ 0006 │ 22 │ 2222 │ 1111 │ M1/T1/1
│ M1 │ MREQ │      │ RD │    │ 0006 │ 19 │ 2222 │ 1111 │ M1/T2/0
│ M1 │ MREQ │      │ RD │    │ 0006 │ 19 │ 2222 │ 1111 │ M1/T2/1
│    │      │ RFSH │    │    │ 0002 │ 19 │ 2222 │ 1111 │ M1/T3/0
│    │ MREQ │ RFSH │    │    │ 0002 │ 19 │ 2222 │ 1111 │ M1/T3/1
│    │ MREQ │ RFSH │    │    │ 0002 │ 19 │ 2222 │ 1111 │ M1/T4/0
│    │      │ RFSH │    │    │ 0002 │ 19 │ 2222 │ 1111 │ M1/T4/1
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1111 │ X/T1/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1111 │ X/T1/1
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1111 │ X/T2/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1111 │ X/T2/1
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1111 │ X/T3/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T3/1 <== result low byte
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T4/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T4/1
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T5/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T5/1
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T6/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T6/1
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 1133 │ X/T7/0
│    │      │      │    │    │ 0002 │ 19 │ 2222 │ 3333 │ Z/T7/1 <== result high byte
```

This time, no clock cycles are wasted. The 16-bit result is only
ready in the very last half cycle of the instruction. Not shown 
here is that the flag bits (H and C) are updated in the opcode fetch
machine cycle of the next instruction (at M1/T3/1).

The relative jump **JR d** performs a regular memory read machine cycle
after the opcode fetch, and then spends 5 more clock cycles to compute
the jump target address:

```
JR d
┌────┬──────┬──────┬────┬────┬──────┬────┬──────┬──────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AB   │ DB │ PC   │ WZ   │
├────┼──────┼──────┼────┼────┼──────┼────┼──────┼──────┤
│ M1 │      │      │    │    │ 0002 │ 00 │ 0002 │ 5555 │ M1/T1/0 <== opcode fetch
│ M1 │ MREQ │      │ RD │    │ 0002 │ 00 │ 0003 │ 5555 │ M1/T1/1
│ M1 │ MREQ │      │ RD │    │ 0002 │ 18 │ 0003 │ 5555 │ M1/T2/0
│ M1 │ MREQ │      │ RD │    │ 0002 │ 18 │ 0003 │ 5555 │ M1/T2/1
│    │      │ RFSH │    │    │ 0002 │ 18 │ 0003 │ 5555 │ M1/T3/0
│    │ MREQ │ RFSH │    │    │ 0002 │ 18 │ 0003 │ 5555 │ M1/T3/1
│    │ MREQ │ RFSH │    │    │ 0002 │ 18 │ 0003 │ 5555 │ M1/T4/0
│    │      │ RFSH │    │    │ 0002 │ 18 │ 0003 │ 5555 │ M1/T4/1
│    │      │      │    │    │ 0003 │ 18 │ 0003 │ 5555 │ M2/T1/0 <== memory ready
│    │ MREQ │      │ RD │    │ 0003 │ 18 │ 0004 │ 5555 │ M2/T1/1
│    │ MREQ │      │ RD │    │ 0003 │ FC │ 0004 │ 5555 │ M2/T2/0
│    │ MREQ │      │ RD │    │ 0003 │ FC │ 0004 │ 5555 │ M2/T2/1
│    │ MREQ │      │ RD │    │ 0003 │ FC │ 0004 │ 5555 │ M2/T3/0
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5555 │ M2/T3/1
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5555 │ X/T1/0  <== extra cycles
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5555 │ X/T1/1
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5555 │ X/T2/0
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5500 │ X/T2/1
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5500 │ X/T3/0
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5500 │ X/T3/1
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5500 │ X/T4/0
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5500 │ X/T4/1
│    │      │      │    │    │ 0003 │ FC │ 0004 │ 5500 │ X/T5/0
│    │      │      │    │    │ 0001 │ FC │ 0004 │ 0000 │ X/T5/1 <== dst addr in WZ
```

The computed target address isn't stored in the **PC** register, but instead
in the internal 16-bit 'helper' register **WZ**. In fact the PC register *never*
contains the actual target address (0000), it switches straight from the address
following the **JR d** instruction (0004) to the address following the
destination address:

```
JR d CONTINUED: NOP at the jump destination (address 0000)
┌────┬──────┬──────┬────┬────┬──────┬────┬──────┬──────┐
│ M1 │ MREQ │ RFSH │ RD │ WR │ AB   │ DB │ PC   │ WZ   │
├────┼──────┼──────┼────┼────┼──────┼────┼──────┼──────┤
│ M1 │      │      │    │    │ 0000 │ FC │ 0004 │ 0000 │ <== PC still 0004!
│ M1 │ MREQ │      │ RD │    │ 0000 │ FC │ 0001 │ 0000 │ <== PC goes right to 0001!
│ M1 │ MREQ │      │ RD │    │ 0000 │ 00 │ 0001 │ 0000 │
│ M1 │ MREQ │      │ RD │    │ 0000 │ 00 │ 0001 │ 0000 │
│    │      │ RFSH │    │    │ 0003 │ 00 │ 0001 │ 0000 │
│    │ MREQ │ RFSH │    │    │ 0003 │ 00 │ 0001 │ 0000 │
│    │ MREQ │ RFSH │    │    │ 0003 │ 00 │ 0001 │ 0000 │
│    │      │ RFSH │    │    │ 0000 │ 00 │ 0001 │ 0000 │
```

TODO DJNZ

TODO JR cc

## Main Quadrant 3 (xx == 11)

<style>
.z80t { border:1px solid black;border-collapse:collapse;padding:5px; }
.z80h { border:1px solid black;border-collapse:collapse;padding:5px;color:black;background-color:Gainsboro }
.z80c0 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:PaleGreen; }
.z80c1 { border:1px solid black;border-collapse:collapse;padding:5px;font-size:80%;font-weight:bold;color:black;background-color:LightPink; }
</style>
<table class="z80t">
<tr class="z80t"><th class="z80h">x=11</th><th class="z80h">z=000</th><th class="z80h">z=001</th><th class="z80h">z=010</th><th class="z80h">z=011</th><th class="z80h">z=100</th><th class="z80h">z=101</th><th class="z80h">z=110</th><th class="z80h">z=111</th></tr><tr class="z80t"><th class="z80h">y=000</th><td class="z80c1">RET NZ</td><td class="z80c0">POP BC</td><td class="z80c0">JP NZ,nn</td><td class="z80c0">JP nn</td><td class="z80c1">CALL NZ,nn</td><td class="z80c1">PUSH BC</td><td class="z80c0">ADD n</td><td class="z80c1">RST 0h</td></tr><tr class="z80t"><th class="z80h">y=001</th><td class="z80c1">RET Z</td><td class="z80c0">RET</td><td class="z80c0">JP Z,nn</td><td class="z80c0">CB prefix</td><td class="z80c1">CALL Z,nn</td><td class="z80c1">CALL nn</td><td class="z80c0">ADC n</td><td class="z80c1">RST 8h</td></tr><tr class="z80t"><th class="z80h">y=010</th><td class="z80c1">RET NC</td><td class="z80c0">POP DE</td><td class="z80c0">JP NC,nn</td><td class="z80c0">OUT (n),A</td><td class="z80c1">CALL NC,nn</td><td class="z80c1">PUSH DE</td><td class="z80c0">SUB n</td><td class="z80c1">RST 10h</td></tr><tr class="z80t"><th class="z80h">y=011</th><td class="z80c1">RET C</td><td class="z80c0">EXX</td><td class="z80c0">JP C,nn</td><td class="z80c0">IN A,(n)</td><td class="z80c1">CALL C,nn</td><td class="z80c0">DD prefix</td><td class="z80c0">SBC n</td><td class="z80c1">RST 18h</td></tr><tr class="z80t"><th class="z80h">y=100</th><td class="z80c1">RET PO</td><td class="z80c0">POP HL</td><td class="z80c0">JP PO,nn</td><td class="z80c1">EX (SP),HL</td><td class="z80c1">CALL PO,nn</td><td class="z80c1">PUSH HL</td><td class="z80c0">AND n</td><td class="z80c1">RST 20h</td></tr><tr class="z80t"><th class="z80h">y=101</th><td class="z80c1">RET PE</td><td class="z80c0">JP HL</td><td class="z80c0">JP PE,nn</td><td class="z80c0">EX DE,HL</td><td class="z80c1">CALL PE,nn</td><td class="z80c0">ED prefix</td><td class="z80c0">XOR n</td><td class="z80c1">RST 28h</td></tr><tr class="z80t"><th class="z80h">y=110</th><td class="z80c1">RET P</td><td class="z80c0">POP AF</td><td class="z80c0">JP P,nn</td><td class="z80c0">DI</td><td class="z80c1">CALL P,nn</td><td class="z80c1">PUSH AF</td><td class="z80c0">OR n</td><td class="z80c1">RST 30h</td></tr><tr class="z80t"><th class="z80h">y=111</th><td class="z80c1">RET M</td><td class="z80c1">LD SP,HL</td><td class="z80c0">JP M,nn</td><td class="z80c0">EI</td><td class="z80c1">CALL M,nn</td><td class="z80c0">FD prefix</td><td class="z80c0">CP n</td><td class="z80c1">RST 38h</td></tr>
</table><br>
