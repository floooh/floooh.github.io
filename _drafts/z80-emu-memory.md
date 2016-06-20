---
layout: post
title: "Emulating the Z80: Memory Management"
---

Before diving into the actual CPU emulation, here's a little blog post
about memory management in 8-bit home computer emulators.

### Bank Switching

From the viewpoint of a CPU, memory access looks very simple: the Z80
could read or write 8- and 16-bit values from and to memory, with an
address space of 64 KByte. For a very simple emulator it would be enough
to represent memory as a simple 64 KByte byte array.

The memory situation in actual home computers was not that simple though. In
the 70s and very early 80s memory was expensive, and home computers couldn't
afford to add the full 64 KByte (the Sinclair ZX80 only had 1 KByte RAM and
4 KByte ROM for instance).

Here's an example of such a simple memory map: the East German Z1013 had 
16 KByte of general RAM at the start of the address range, a 1 KByte video
memory, and a 2 KByte system ROM:

```                                     
0000           4000            8000            C000   EC00 F000
+--------------+----------------------------------------+-+-+--+
|     RAM      |=============== unused =================| | |==|
+--------------+----------------------------------------+-+-+--+

16 KByte RAM from 0x0000 to 0x4000
 1 KByte vidmem from 0xEC00 to 0xEFFF
 2 KByte system ROM from 0xF000 to 0xF7FF
```

A huge part of the address space from 0x4000 to 0xEBFF, and from 0xF800 to
0xFFFF was 'empty'.

Then during the 80's RAM quickly got cheaper, and home computers were sold
with more than the 64 KByte RAM.

For example here's the East German KC85/4 which came with 128 KByte RAM
(64 KByte alone for video memory) and 20 KByte ROM:

```
0000          4000            8000            C000     F000
+-------------+---------------+---------------+--------+-------+
|    RAM0     |     RAM4      |      VID0     |  BASIC | CAOS  |
+-------------+---------------+---------------+----+---+-------+
                              |      VID1     |CAOS|
                              +---------------+----+
                              |      VID2     |
                              +---------------+
                              |      VID3     |
                              +---------------+
                              |     RAM8-0    |
                              +---------------+
                              |     RAM8-1    |
                              +---------------+
```

Note the 6-deep stack of 16 KByte memory banks at address 0x8000, 4 of those
were video memory (2 sets of pixel/color banks, one of those displayed and one
hidden), and 2 more general RAM banks. Since the operating system (called CAOS)
didn't fit into 8 KByte anymore, a small 4 KByte ROM bank was hidden at address
0xC000 behind the BASIC ROM.


To make the additional RAM accessible by the CPU, **bank switching** was used
which allowed to switch 'memory banks' in and out of the CPU-visible
address range. Memory banks that are currently 'switched out' kept their
content but could not be accessed by the CPU. Bank swichting was very fast 
since no memory was copied, only the address bus of the CPU was 're-wired'
to different memory chips (at least that's how imagine it as a total
hardware noob).

### Memory Mapped I/O

Some CPUs (most notably the 6502) communicate with 'peripheral devices' (for
instance custom chips) by writing to or reading from special memory locations
which are not backed by normal memory but mapped to device I/O registers.

TO BE CONTINUED....







