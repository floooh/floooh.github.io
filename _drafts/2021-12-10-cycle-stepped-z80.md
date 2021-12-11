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
emulator would execute one full instruction before it could return
to the caller.  While this doesn't prevent the creation of 'cycle-correct'
computer system emulators (because the CPU emulator would invoke a 'tick callback'
multiple times while executing a single instruction), switching the
CPU emulator to a cycle-stepped model allows a more straight-forward
'whole system' emulation because the CPU no longer has a special controller
role in a system emulator, but instead is just an ordinary chip that's
ticked along with other the other chip emulators.

The simplicity of the new emulator is best shown in a code example which
runs through a small machine code program to add two numbers:

```c++
#include <stdio.h>
#define CHIPS_IMPL
#include "z80.h"


int main() {
    // 64 KB memory with test program at address 0x0000
    uint8_t mem[(1<<16)] = {
        0x3E, 0x02,     // LD A,2
        0x06, 0x03,     // LD B,3
        0x80,           // ADD A,B
        0x00,           // NOP...
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

Here's the same code on [Compiler Explorer](https://www.godbolt.org/z/6aTKo4vbM).

The emulator only has two important API functions:

```uint64_t z80_init(z80_t* cpu)```

...this initializes a Z80 instance and returns an 'ignition' pin mask which is passed into the first tick 
function call:

```uint64_t z80_tick(z80_t* cpu, uint64_t pins)```

...the tick function executes exactly one clock cycle and then returns to the caller 
with a 'pin bitmask'. The pin bitmask is used to communicate with the 'outside world'
just like the input/output pins on a real CPU.

The caller inspects the returned pin mask (for instance to check if the CPU wants to read
or write memory), modifies the pin mask accordingly, and feeds it back into the next
call of ```z80_tick()```. And that's it, no more 'system tick callback', or
'trace callback' for debugger support like in the old Z80 emulator.

## Basic Implementation Ideas

Like in the previous emulator, instruction decoding happens in a big switch-case 
statement, but instead of one case-branch per opcode there's now one case branch
per clock cycle.

In the new emulator, the decoder switch-case statement looks much more like a program on its
own with complex control flow like unconditional and conditional branches and loops.

A simple sequence where each call to the tick function executes the next step looks like this:

```c++
void tick(state_t* state) {
    switch (state->step++) {
        case 0: ... break;
        case 1: ... break;
        case 2: ... break;
        ...
    }
}
```
Calling such a tick function over and over would first step through all the case branches until
there are no more branches left or the step counter overflows.

An unconditional branch would simply set the step counter to a different value, for instance
here the step counter is reset to zero at the end of the sequence, implementing an 
infinite loop:
```c++
void tick(state_t* state) {
    switch (state->step++) {
        case 0: ... break;
        case 1: ... break;
        case 2: ... break;
        case 3: state->step = 0; break;
    }
}
```
Calling the tick function will now execute the steps 0..3 forever: [0, 1, 2, 3, 0, 1, 2, 3, ...].

Of course branches can also jump forward to skip steps:
```c++
void tick(state_t* state) {
    switch (state->step++) {
        case 0: ... break;
        case 1: state->step = 3; break;
        case 2: ... break;
        case 3: state->step = 0; break;
    }
}
```
Step 2 will now always be skipped, resulting in [0, 1, 3, 0, 1, 3, ...].

...and conditional branches only modify the step counter when a condition is true:
```c++
void tick(state_t* state) {
    switch (state->step++) {
        case 0: state->cond = !state->cond; break;
        case 1: if (state->cond) { state->step = 3; } break;
        case 2: ... break;
        case 3: state->step = 0; break;
    }
}
```

Because ```state->cond``` is now flipped in step 0, step 2 will be skipped in every other 'run' of the program: [0, 1, 3, 0, 1, 2, 3, 0, 1, 3, ...].

Such a simple switch-case state machine can be used to split *any* program logic into a unique
steps, and in fact that's exactly what many compilers do under the hood for their async-await
code.

Code that needs to be executed on each call of the tick function can be placed in front (prologue) or after (epilogue) the switch-case statement:

```c++
void tick(state_t* state) {
    prologue();
    switch (state->step++) {
        case 0: state->cond = !state->cond; break;
        case 1: if (state->cond) { state->step = 3; } break;
        case 2: ... break;
        case 3: state->step = 0; break;
    }
    epilogue();
}
```

And finally, in C we can "break into" different epilogue code blocks using the
good o'le goto statement:
```c++
void tick(state_t* state) {
    prologue;
    switch (state->step++) {
        case 0: state->cond = !state->cond; goto ep_1;
        case 1: if (state->cond) { state->step = 3; } goto ep_1;
        case 2: ...; goto ep_2;
        case 3: state->step = 0; goto ep_1;
    }
ep_1:
    epilogue_1();
ep_2:
    epilogue_2();
}
```
Now the steps 0, 1 and 3 will run ```epilogue_1()``` and ```epilogue_2()``` on function
exit, but step 2 will only execute ```epilogue_2()```.

With the above core ideas it's now possible to encode Z80 instructions into
switch-case 'microcode'.

For example **LD HL,nnnn** instruction which loads a 16-bit immediate value into
the register pair HL. Execution of the instruction takes 10 clock cycles, split
into 3 machine cycle groups:

- opcode fetch and refresh (4 clock cycles)
- read low byte (3 clock cycles)
- read high byte (3 clock cycles)

...encoded into a switch-case sequence (as simplified pseudocode, abus and dbus means 'address bus' and 'data bus'):

```c++
uint64_t tick(z80_t* cpu, uint64_t pins) {
    switch (cpu->step) {
        // opcode fetch + refresh machine cycle
        case 0: pins = (M1|MREQ|RD) | set_abus(cpu->PC++); break;
        case 1: cpu->opcode = get_dbus(pins); break;
        case 2: pins = (RFSH|MREQ) | set_abus(...); break;
        case 3: break;

        // first read machine cycle
        case 4: break;
        case 5: pins = (MREQ|RD) | set_abus(cpu->pc++); break;
        case 6: cpu->L = get_dbus(pins); break;

        // second read machine cycle
        case 7: break;
        case 8: pins = (MREQ|RD) | set_abus(cpu->pc++); break;
        case 9: cpu->H = get_dbus(pins); cpu->step = 0; break;
    }
}
```

This looks pretty good already, but there's a chicken-egg-problem: the
opcode fetch machine (the first four steps) doesn't do anything useful.
Even though it loads an opcode byte, the opcode is ignored and the CPU
will only ever execute **LD HL,nnnn** instructions. Currently there's no
way to add additional instruction microcode programs to the switch case
statement.

The solution is to split instructions into a shared opcode fetch machine
cycle which is the same for all instructions, and then do a branch 
to the remaining steps in an instruction somewhere else in the big
switch-case statement.

At the end of the shared opcode fetch machine cycle, a table lookup
is performed with the opcode as index which yields the first step
of the actual instrucion 'payload' (check out ```case 3:```):


```c++
uint64_t tick(z80_t* cpu, uint64_t pins) {
    switch (cpu->step) {
        // opcode fetch + refresh machine cycle
        case 0: pins = (M1|MREQ|RD) | set_abus(cpu->PC++); break;
        case 1: cpu->opcode = get_dbus(pins); break;
        case 2: pins = (RFSH|MREQ) | set_abus(...); break;
        case 3: cpu->step = cpu->opstep[cpu->opcode];

        ...

        // somewhere else in the big switch-case:
        // first read machine cycle of LD HL,nnnn
        case N+0: break;
        case N+1: pins = (MREQ|RD) | set_abus(cpu->pc++); break;
        case N+2: cpu->L = get_dbus(pins); break;

        // second read machine cycle of LD HL,nnnn
        case N+3: break;
        case N+4: pins = (MREQ|RD) | set_abus(cpu->pc++); break;
        case N+5: cpu->H = get_dbus(pins); cpu->step = 0; break;
        
        ...
    }
}
```

Note how the last clock loops back to step 0 to load the next opcode
byte. In general, that's how the emulation loop works, but there's one
last problem: most instructions in the Z80 instruction set take 4 cycles
to execute, but this doesn't fit into the above execution pattern.

Since the opcode fetch machine cycle already takes 4 clock cycles, there's
no clock cycle left to execute the instruction 'payload'.

Let's take the **ADD A,B** instruction as example, this *should* look likt his:

```c++
uint64_t tick(z80_t* cpu, uint64_t pins) {
    switch (cpu->step) {
        // opcode fetch + refresh machine cycle
        case 0: pins = (M1|MREQ|RD) | set_abus(cpu->PC++); break;
        case 1: cpu->opcode = get_dbus(pins); break;
        case 2: pins = (RFSH|MREQ) | set_abus(...); break;
        case 3: cpu->step = cpu->opstep[cpu->opcode];

        ...

        // ADD A,B payload:
        case N+0: cpu->a=add8(cpu->a,cpu->b); cpu->step=0; break;
        
        ...
    }
}
```

This would take 5 clock cycle, not the required 4.

The solution is to use the same trick as a real-world Z80: overlap the
execution of the last instruction with the opcode fetch of the next instruction
by merging the last clock cycle of an instruction with the first clock cycle of
the next opcode fetch, note how the old ```case 0:``` step has been removed and
merged with the ADD A,B 'payload' step:

```c++
uint64_t tick(z80_t* cpu, uint64_t pins) {
    switch (cpu->step) {
        // last 3 clock cyles of opcode fetch + refresh machine cycle
        case 0: cpu->opcode = get_dbus(pins); break;
        case 1: pins = (RFSH|MREQ) | set_abus(...); break;
        case 2: cpu->step = cpu->opstep[cpu->opcode];

        ...
        // ADD A,B payload:
        case N+0: 
            cpu->a=add8(cpu->a,cpu->b); 
            pins = (M1|MREQ|RD) | set_abus(cpu->PC++);
            cpu->step = 0;
            break;
        ...
    }
}
```

...it looks a bit nicer when wrapping the actions of the first opcode fetch cycle into a macro:

```c++
#define _fetch() {pins=(M1|MREQ|RD)|set_abus(cpu->PC++);cpu->step=0}

uint64_t tick(z80_t* cpu, uint64_t pins) {
    switch (cpu->step) {
        // last 3 clock cyles of opcode fetch + refresh machine cycle
        case 0: cpu->opcode = get_dbus(pins); break;
        case 1: pins = (RFSH|MREQ) | set_abus(...); break;
        case 2: cpu->step = cpu->opstep[cpu->opcode];

        ...
        // ADD A,B payload and first clock cycle of next opcode fetch
        case N+0: cpu->a=add8(cpu->a,cpu->b); _fetch(); break;
        ...
    }
}
```

With this overlap-trick the timing for 4-cycle instruction is working fine now. You might be 
wondering now how the very first instruction starts executing now. The trick is that the ```z80_init()```
function doesn't initialize the ```cpu->step``` to 0, but instead to the overlapped 'payload and fetch' clock
cycle of the NOP instruction. That way the CPU emulation is properly 'ignited' at the first call
to the ```z80_tick()``` function.

The only thing to figure out now is how to 'draw the rest of the f*cking owl'.



Code generation:

    - YAML desc file
    - python script
    - C header template

Implementation details:

    - shared opcode fetch machine cycle
    - opcode lookup table
    - overlapped exec/fetch cycle
    - wait cycles
    - instructions with variable timing
    - DD/FD opcode fetch
    - DD/FD HL/IX/IY renaming
    - ED instruction block
    - CB instruction block
    - interrupt detection & handling
    - differences to real CPU
        - pins only active for 1 clock cycle
        - wait vs read vs write
        - overlapped execution only 1 clock cycle
        - 


Roads not taken:

    - fixed opcode slot ranges (like in 6502 emu)
    - 'execution bit pipeline'
    - 