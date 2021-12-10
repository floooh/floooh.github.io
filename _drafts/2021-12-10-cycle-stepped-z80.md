---
layout: post
title: A new cycle-stepped Z80 emulator
---

I finally got around rewriting the [Chips Project](https://github.com/floooh/chips) Z80 emulator to a
'cycle-stepped' execution model. The idea has been rolling around in my head
since at least 2019 when I gave the
[6502 emulator the cycle-stepped treatment](https://floooh.github.io/2019/12/13/cycle-stepped-6502.html).

'Cycle-stepped' means that the CPU emulator state can be stepped forward in
single clock cycles. Previously, the Z80 emulator was 'instruction stepped':
Even if the user asked the emulator to only execute one clock cycle, the
emulator would run until the end of the next instruction before it could return
to the caller.  While this doesn't prevent the creation of 'cycle-correct'
computer system emulators (because the CPU emulator would invoke a 'tick callback'
multiple times while executing a single instruction), switching the
CPU emulator to a cycle-stepped model can enable a more straight-forward
'whole-system-emulation' because the CPU is now 'just another chip' in the
computer system emulation.

The simplicity of the new emulator is best shown in a code example which
runs through a small machine code program to add two numbers:

```c
#include <stdio.h>
#define CHIPS_IMPL
#include "z80.h"

int main() {
    // 64 KB memory with test program at address 0x0000
    uint8_t mem[(1<<16)] = {
        0x3E, 0x02,     // LD A,2
        0x06, 0x03,     // LD B,3
        0x80,           // ADD A,B
    };

    // initialize Z80 emu and execute some clock cycles 
    z80_t cpu;
    uint64_t pins = z80_init(&cpu);
    for (int i = 0; i < 20; i++) {
        
        // tick the CPU
        pins = z80_tick(&cpu, pins);

        // handle memory read or write access
        if (pins & Z80_MREQ) {
            if (pins & Z80_RD) {
                Z80_SET_DATA(pins, mem[Z80_GET_ADDR(pins)]);
            }
            else if (pins & Z80_WR) {
                mem[Z80_GET_ADDR(pins)] = Z80_GET_DATA(pins);
            }
        }
    }

    // register A should now be 5
    printf("\nRegister A: %d\n", cpu.a);
    return 0;
}
```

Here's the same code on [Compiler Explorer](https://www.godbolt.org/z/n9qsYG98a).

The emulator only has two important API functions:

```uint64_t z80_init(z80_t* cpu)```


...this initializes an emulator instance and returns an 'ignition' pin mask which must into the first call of the tick function:

```uint64_t z80_tick(z80_t* cpu, uint64_t pins)```

...which executes exactly one clock cycle and then returns to the caller.
z80_tick() returns a 'pin mask' which is the only interface to the outside world,
just like the input/output pins on a real CPU.

The caller inspects the returned pin mask (for instance to check if the CPU wants to read
or write memory), modifies the pin mask accordingly, and feeds it back into the next
call of ```z80_tick()```. And that's it, no more callbacks that are called
from inside the CPU emulation, which also means that the CPU emulation now no
longer plays a special 'controller role' in a system emulation, instead the
CPU is now a chip like any other in the system.

