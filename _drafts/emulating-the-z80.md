---
layout: post
title: Emulating the Z80
---

TL;DR: Interesting tidbits and things to consider when writing a Z80 emulator,
on the example of the [YAKC](http://github.com/floooh/YAKC) Z80 emulator class.

### Broad Strokes

These are the broad functional areas of a Z80 emulator that need to
be considered:

- Z80 CPU state (registers and internal state bits)
- memory access
- instruction fetch, decode and execute
- input/output (how the CPU talks to the outside world)
- interrupt handling (how the CPU handles high-priority events from the outside world)
- implementing undocumented behaviour
- testing and performance tuning

### Z80 CPU State

The Z80 has eight 8-bit registers that can also be accessed as 16-bit
register pairs, in the emulator these are implemented as anonymous unions,
for instance the register pair B,C looks like this:

```cpp
class z80 {
    ...
    union {
        struct { uint8_t C, B; };
        uint16_t BC;
    };
    ...
};
```

I didn't care about big-endian CPUs, on those, the order of 'uint8\_t C,B'
would need to be reversed.

This allows to access the access both the 8-bit registers or the
16-bit pairs directly:

```cpp
    z80 cpu;
    cpu.B = b;
    cpu.C = c;
    uint16_t bc = cpu.BC;
```

All the other registers and internal state are implemented as normal 
uint8\_t or uint16\_t class members.

### Memory Access

The emulator only needs to read and write 8- or 16-bit values from and
to memory, so theoretically a simple 64 KByte flat byte array would do. 
In practice however, many 8-bit home computers had the ability to work
with more than 64 KByte RAM by mapping memory banks in and out of the 
CPU-visible address space.

Certain memory ranges also need to be marked as read-only for ROM banks.
Trying to write to those areas will not be a fatal error, but the
underlying values in memory will not be modified.

The Z80 emulator delegates memory management into its own class, aptly called
'memory'. This works a little bit like MMUs (memory management units) in
later CPUs, 'virtual memory' is mapped to 'physical memory' through
memory pages, and the translation from a virtual address to a physical
address happens through a page table lookup. In the context of the emulator,
'virtual memory' is the 64-KByte address range which is visible to the
CPU, and 'physical memory' is the backing memory provided by the host 
system.

A page in the memory class is described by a pointer to host memory and
a writable flag:

```cpp
struct page {
    uint8_t* ptr;
    bool writable;
}
```

The page size is currently 1 KByte. It started at 16 KByte, and then with each
new emulated system had to be reduced to 8, 4 and finally 1 KByte, since the
Z1013 and Z9001 computers have small 1 KByte video memory areas.

With a 1 KByte page size (== 2^10, or 0x400), an 8-bit memory read operation
which goes through a page-table lookup looks like this:

```cpp
uint8_t memory::r8(uint16_t addr) const {
    return this->pages[addr>>10].ptr[addr&0x3FF];
}
```

The corresponding write operation needs to check the writable bit:

```cpp
void memory::w8(uint16_t addr, uint8_t b) const {
    const auto& page = this->pages[addr>>10];
    if (page.writable) {
        page.ptr[addr&0x3FF] = b;
    }
}
```

The memory class also implements the bank switching logic with currently 4
layers, meaning that up to 4 memory banks mapped to the same address can be
stacked on top of each other. The bank switching logic simulates the modular
memory bank management in the more sophisticated 8-bit computers. As an
example, several 16-KByte memory banks could be mapped to the same address (for
instance from 0x4000 to 0x7FFF), but only the highest priority active memory
bank would be visible to the CPU. Switching the highest priority bank to
'inactive' would preserve its memory content, but it would no longer be mapped
to the CPU-visible address range, instead the memory bank with the
second-highest priority would be visible to the CPU.

Some of the more sophisticated 8-bit systems like the KC85/4 were able 
to manage up to 4 MByte RAM through memory bank switching.

Unfortunately the page-table indirection is a big performance hit of about
30% for the emulator. More on that later in the Performance section.

### Instruction fetch, decode and execute

This is by far the biggest part of the Z80 emulator, the most performance-
critical, and with 3 rewrites also the part that saw the most change.

At the highest level, the emulator provides a 'step()' method that loads the
next instruction from memory, 'executes' it and returns the number of CPU
cycles taken. The returned instruction cycle count is used to synchronize the
CPU emulator with the real-world clock, and with other parts of a complete
computer system emulator like attached support chips and video/audio output.

At the highest level, the step() method looks quite simple:

```cpp
uint32_t z80::step() {
    // fetch next instruction byte from the instruction pointer
    // location (PC) and increment the instruction pointer
    uint8_t opcode = mem.r8(PC++);

    // ???
    ...
    return cycles;
}
```

The '???' is the interesting part of course, and there's also a bit
more to do then simply reading the next instruction byte:

- the memory refresh register R needs to be updated for each instruction
fetch
- if interrupts were enabled during the last instruction, set the interrupt
state bits IFF1 and IFF2
- more opcode bytes may have to be fetched in case of multi-byte instructions

For the actual decode-and-execute part I have gone through different rewrites:

#### Attempt 1: Dumb code-gen, giant switch-case

This was the first straight-forward but tedious attempt, just take the [Z80
User Manual](http://www.z80.info/zip/z80cpu_um.pdf), and implement the
instructions described there one by one. I decided to use code generation,
write a python script which spits out a C++ source file containing a single
step() method made of a giant switch-case statement.

Every branch in the switch-case statement would implement a single instruction
and return a hardwired cycle count. For multi-byte instructions, the switch-case
was nested, certain lead-byte values (0xCB, 0xED, 0xDD and 0xFD) would
fetch another instruction byte and based on its value branch into another
giant sub-switch-case.

In parallel to the code generation I wrote a huge unit-test source
file which tested the resulting register, memory and flag-bit state for
each new instruction chunk I implemented.

This was long and tedious work. Every evening I would spend 2 or 3 hrs 
implementing a few pages of the Z80 User Manual for about 3 weeks. The
test source code alone was over 2700 lines of code.

But in the end the rigorous testing discipline worked out quite well, apart
from a small flag computation error in the 16-bit arithmetic instructions I got
the KC85/3 system ROM booting up almost immediately. There were a few minor
bugs left which I found out about only much later when some games didn't work
properly, and none of the undocumented parts were implemented yet, but it was
enough to continue working on the other emulator parts.

This 'dumb giant switch-case' approach was fast out of the box, on my
mid-2014 MacBook Pro with 2.8 GHz Intel i5 I could emulate an emulated
Z80 running at around 1.2 GHz.

#### Attempt 2: No code-gen, algorithmic decode

I wasn't very happy with the previous 'dumb' code generation approach,
since the code-gen python script was nearly as long as the 
generated code (somewhere between one and two-thousand lines of code). 

There is a better 'algorithmic' approach to instruction decoding which 
looks at the instruction bit patterns, this is described in detail
[here](http://www.z80.info/decoding.htm).

The gist of this is that an instruction byte can be split into 3 bit groups,
and those bit groups define the type of instruction, and source and
destination operands. This simple idea comes with a lot of exceptions,
so that the resulting code is still pretty big and messy, but
it is much cleaner and smaller than the 'dumb switch case' approach.

My next attempt was to implement this algorithmic approach directly in
C++ code, getting rid completely of the python code generation. Since there
was so much less code I was hoping that performance would even be slightly
better than the giant switch-case because of better instruction cache
usage, and even if the performance would be a little bit worse I would
have stayed with this approach because of the simpler code. 

But the real-world performance was disappointing. The algorithmic decoder
only reached about 60% of the dumb switch-case.

#### Attempt 3: Algorithmic code-gen, giant switch-case

Heavy-hearted I gave up on the second attempt and started to combine the
two previous ways: there would be a code-generation python script which
generated a giant nested switch-case decoder as before, but this time
the python script would use the new algorithmic approach and thus would
be much smaller than the old 'dumb' code generation script. 

This attempt works very well. It gives me the great performance of the
nested switch-case statement, coupled with a much simpler and easier to maintain
python code generator, which later simplified the whole
undocumented-behaviour implementation a lot.

#### Flags bits and other tricky things

The whole instruction decoding and execution work was fairly straight 
forward except for a few parts that are worth mentioning.

First, **flag bit computation**:

Most instructions that are not simple load/store operations modify the
flag bits in the F register. The Z80 has 6 documented flag bits:

- **S (bit 7)**:  sign flag, set for negative results
- **Z (bit 6)**:  zero flag, set if result is zero
- **H (bit 4)**:  half carry flag: set if a carry/borrow happened into/from result bit 4
- **P/V (bit 2)**: parity/overflow, depending on instruction type, number of 1-bits
in result, or whether an arithmetic overflow happened
- **N (bit 1)**: add/sub flag, set if the last instruction was a subtraction
- **C (bit 0)**: carry flag, set if an overall carry/borrow happened

The bit position of some of the flag bits is interesting: for instance
the **sign flag** is at bit 7, the same position of the sign bit in a byte.

This makes computing the sign flag from a result simple:

```cpp
    // CF is defined as (1<<7)
    f = res & CF;
```

If the result 'res' is negative, the flags value will have the sign bit
set.

The half-carry-flag H is at an equally interesting position at bit 4, but
the computation is a bit more tricky:

Half-carry is set if a carry into or borrow from bit 4 happened during the
last operation. There's a nice trick to compute the status of the 
half-carry flag both for addition and subtraction which looks like this:

```cpp
    // HF is (1<<4)
    uint8_t res = A + val;    // could also be 'A - val'
    f = (A^val^res)&HF;
```

The half-carry flag is set if bit 4 is either set exactly once in both
summands and the result, or in all three:

```
    0^0^0 => 0
    0^0^1 => 1
    0^1^0 => 1
    0^1^1 => 0
    1^0^0 => 1
    1^0^1 => 0
    1^1^0 => 0
    1^1^1 => 1
```

Let's look how this works in detail by looking at all possible
combinations for bit 3 and 4 of both summands and the result, and
when a carry from bit 3 to bit 4 would happen:

```
    00 + 00 => 00
    00 + 01 => 01
    00 + 10 => 10
    00 + 11 => 11

    01 + 00 => 01
    01 + 01 => 10    carry from bit 3 to 4
    01 + 10 => 11
    01 + 11 => 00    carry from bit 3 to 4

    10 + 00 => 10
    10 + 01 => 11
    10 + 10 => 00
    10 + 11 => 01    

    11 + 00 => 11
    11 + 01 => 00    carry from bit 3 to 4
    11 + 10 => 01
    11 + 11 => 10    carry from bit 3 to 4
```

Note by just looking at bit 4 (on the left side of each 2-bit number)
that in each case where a carry happens, exactly one of the 3 bits
is set, or in the last case all 3 bits are set, which is the same
as a^b^c == 1.

For the **carry flag C** the computation is much simpler, this is set if
a carry or borrow from bit 7 to an imagined bit 8 would happen, but
since the starting state of this imagined bit 8 is always 0, we only
need to check if bit 8 of the result of an addition or subtraction
extended to uint16_t or uint32_t would be set (we can't do the 
operation in uint8_t since we need the state of bit 8).

The next flag bit that's a bit non-trivial to compute is the
**overflow flag**.

(TODO: CONTINUE)










