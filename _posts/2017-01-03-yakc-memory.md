---
layout: post
title: "YAKC Emu: The Memory System"
---

This is the second blog post in my little YAKC Emulator internals series which
looks at the emulator's memory system, pretty much the lowest level part of the
emulator.

Here is the emulator's [github repository](https://www.github.com/floooh/yakc),
and here is the [asm.js version](https://floooh.github.com/virtualkc)

#### Memory System Requirements

8-bit CPUs like the Z80 or MOS-6502 typically have a 16-bit address bus and 8-bit
data bus, meaning that the CPU has a 64 KByte flat address space and can read
or write one byte at a time (the Z80 also has instructions that can read or
write 16-bit words, but these still require two 8-bit memory accesses internally).

In theory, the emulator memory system could just be a flat 64 KByte memory
buffer, alas the 8-bit home computers weren't that simple:

- In the late 70's and early 80's, 64 KByte was a lot of memory and too
  expensive for many home computers, so many machines only came with 16
  KByte RAM or even less. This means there are holes in the address space which
  weren't backed by real memory, but an emulator still needs to return
  'something' when the CPU reads from those unmapped memory areas (usually
  the value returned for unmapped memory is 0xFF).

- Not all memory is writable, the operating system usually lived in ROM chips,
  so an emulator memory system must be able to treat memory areas as read-only.

- Starting in the early- to mid-80's, RAM became cheap enough so that home computers
  started to have the full 64 KByte of RAM in addition to the operating system
  ROMs, or even had 128 KByte RAM built in. Expansion modules could add even
  more RAM.  But since the CPU still could only see 64 KBytes of memory at a
  time, some sort of memory management hardware was necessary which could
  map 'memory banks' in and out of the CPUs address space.

- Some systems (like the Amstrad CPC and C64) had 'RAM-behind-ROM' memory
  areas, where a ROM bank and a RAM bank was mapped to the same address range,
  and read operations on an address would access the ROM bank, while write
  operations (which would be useless on ROM anyway) 'poke through the
  ROM' and write to the RAM bank.

- Some home computer systems had 'slow memory' areas, usually video
  memory, where the CPU doesn't have exclusive access but needs to share the
  memory with other devices (such as the video controller). At least on the Z80
  access to such 'contended memory' areas may add 'wait states' to synchronize
  memory access with other devices, increasing the instruction cycle count (I
  don't know yet whether the 6502 has a similar feature).

- And finally there's an important difference between the Z80 and 6502 when
  communicating with support chips and other hardware devices. The Z80 has a
  dedicated set of instructions for this (the IN/OUT instructions). When those
  special I/O instructions were executed, the 16-bit address-bus wasn't used to
  access memory, but as a special 'device address'. This essentially means the
  Z80 had a whole second 16-bit address space reserved for talking to hardware
  devices.  The 6502 on the other hand used 'memory-mapped I/O', communication
  with I/O devices was performed by writing to or reading from special memory
  locations.  The advantage of memory-mapped-I/O is that all the powerful
  addressing modes of the normal memory load/store instructions can be used,
  the disadvantage is that precious memory address ranges are lost because they
  have to be reserved as I/O space.

So the list of requirements for an 8-bit emulator memory system is:

- read and write 8-bit values at 16-bit addresses
- support unmapped 'holes' in the address space
- mark address ranges as read-only
- support memory-bank-switching
- support 'RAM-behind-ROM' access
- for 6502, memory-mapped-IO must be supported
- for Z80, memory-access wait states must be supported

The YAKC memory system currently supports all requirements except the last two,
there is no 6502 support at all yet, and for Z80 systems, the memory
system currently doesn't support additional wait states when accessing certain
memory areas.

#### The YAKC::memory class

The whole YAKC memory system is implemented in a single class, aptly named
**'memory'**. The memory class doesn't *own* any host system memory, it just
performs a mapping from a 16-bit address to an 'externally owned' host-system
memory location and reads from or writes to this address.

To make this address mapping somewhat performant, a page-table is used, not
unlike page-tables in modern CPUs to translate virtual memory addresses to
physical addresses.

#### The Page Table

The YAKC memory class divides the 64 KByte address space into 64 memory pages of 1
KByte each. The 1 KByte page size is a requirement for some of the currently
emulated computers, I actually started with a 16 KByte page size but had to go down
to 8, 4 and finally 1 KByte as I added more emulated computer systems.

A memory page item in the page table is just 2 pointers, one for read-access
and one for write-access:

```cpp
struct page {
    const uint8_t* read_ptr;
    uint8_t* write_ptr;
};
```

The page struct also defines constants related to the page size which I omitted above
for better readability:

```cpp
    static const int shift = 10;            // 1<<10 == 1024 == 1 KByte
    static const uint16_t size = 1<<shift;  // the page size (1 KByte)
    static const uint16_t mask = size - 1;  // page-offset bit mask
```

The page table is just an array of page items with 64 entries (or 64k divided by the page size):

```cpp
    static const int num_pages = (1<<16) / page::size;
    page page_table[num_pages];
```

The 2 pointers per page support all the required mapping scenarios:

- **normal RAM access**: both pointers are valid and point to the same location
- **ROM access**: the write\_ptr is a nullptr, the read\_ptr is valid
- **RAM-behind-ROM**: both pointers are valid, but point to different locations
- **unmapped pages**: the read\_ptr is valid and points to a special 1 KByte
  memory area with 0xFF values, and the write\_ptr is a nullptr

To read or write 8-bit values the following 2 methods exist:

```cpp
    /// read an unsigned byte from memory
    uint8_t r8(uint16_t addr) const;
    /// write an unsigned byte to memory
    void w8(uint16_t addr, uint8_t val);
```

These 2 methods are the hot code paths of the memory system and must be very
fast. They are currently implemented like this:

```cpp
inline uint8_t
memory::r8(uint16_t addr) const {
    return this->page_table[addr >> page::shift].read_ptr[addr & page::mask];
}

inline void
memory::w8(uint16_t addr, uint8_t val) {
    const auto& page = this->page_table[addr >> page::shift];
    if (page.write_ptr) {
        page.write_ptr[addr & page::mask] = val;
    }
}
```

Note the shift and mask operations to map the 16-bit address to the host system
address. The top 6 bits of the 16-bit address form the page table index to
access the right memory page, the right-shift operation shifts those upper 6
bits into the page-index range of 0..63. The lower 10 bits of the 16-bit
address are the byte index into the 1 KByte memory page, and the '& page::mask'
isolates those lower 10 bits and gets rid of the upper 6 bits.

The read\_ptr is guaranteed to be valid, so a separate pointer check is not needed
when going through the read\_ptr (but it is needed for the write\_ptr in case the
page defines a ROM mapping).

The page table is initialized by mapping continuous host memory areas to 16-bit
addresses via the method memory::map():

```cpp
/// map a host memory range to a 16-bit address
void map(int layer, uint16_t addr, uint32_t size, uint8_t* ptr, bool writable);
```

'size' must be a multiple of the page size, this is checked with an assert.

If 'writable' is false, both the page read- and write-pointers are set to
the 'ptr' parameter, otherwise only the read-pointer is assigned, and
the write-pointer is set to nullptr.

For the 'RAM-behind-ROM' scenario there's a second mapping method which
directly allows to define different pointers for the readable and writable
memory chunk:

```cpp
/// map different read/write pointers
void map_rw(int layer, uint16_t addr, uint32_t size, uint8_t* read_ptr, uint8_t* write_ptr);
```

Two things might appear a bit strange:

- what's the 'layer' parameter for?
- why is the 'size' parameter 32-bit?

Second question first: size is 32-bit to allow mapping 64 KBytes at once:

```cpp
    // a 64 KByte array
    uint8_t host_memory[0x10000]
    // map those 64 KByte as RAM to the entire address range
    mem.map(0, 0x0000, 0x10000, host_memory, true);
```

The number 0x10000 simply doesn't fit into 16 bits. I could have used 0x0000 as
a special case but that wouldn't be very intuitive.

#### Page Table Layers

So what's that 'layer' parameter about? 

The memory class has a number of secondary 'layer page tables' in addition to
the primary page table which defines the CPU-visible mapping, and the layer
parameter is simply an index into this array of secondary page tables.  I have
added those layered page tables to simplify some complex mapping scenarios
where memory banks are stacked behind each other and those banks are managed by
independent components in an emulated home computer (for instance an expansion
module system).

The expansion module system could map all its memory banks into layer 1, while
the base computer system only uses memory layer 0. After the memory mapping
changes, the primary page table (which defines the CPU-visible mapping) needs
to be updated from the secondary layer page tables. If the same page table
slot in different layers has a valid mapping, the layer with the lower layer
index 'wins' and provides the CPU-visible page.

Here's a visual example: 

```
                +--------+--------+--------+--------+
Layer 1         |11111111|11111111|11111111|11111111|
                +--------+--------+--------+--------+
                +--------+--------+        +--------+
Layer 0         |00000000|00000000|        |00000000|
                +--------+--------+        +--------+

                +--------+--------+--------+--------+
CPU visible     |00000000|00000000|11111111|00000000|
                +--------+--------+--------+--------+
```

On layer 0, three memory pages are mapped with a hole in the 3rd slot, and
behind it on layer 1 all four memory pages are mapped. The CPU sees the 3
memory pages of layer 0 since layer 0 has a higher priority than layer 1. But
on the 3rd slot, the memory page at layer 1 'peeks through' into the
CPU-visible address space since layer 0 has an unmapped hole there.

Every time the memory mapping configuration is changed, the primary page table
needs to be updated with the CPU-visible layer pages, which is still faster
than doing the layer lookup in each memory access.

The current layered page table system with separate read- and write-pointers per page
implements all requirements except the last two: wait states for the Z80 and memory-mapped-IO
for the 6502.

How would those be implemented?

#### What If: Memory Mapped IO for the 6502 CPU

Instead of associating host memory locations with a 16-bit address range, a
function pointer would be 'mapped', and whenever a read or write on a 16-bit
address within that range happens, this function would be called.

The function prototype would need to look like this in order to handle read and
write operations through the same callback function:

```cpp
uint8_t mem_io(uint16_t addr, bool write, uint8_t in_val);
```

*'addr'* would be the 16-bit memory address, *'write'* would be true for a
write-operation, and false for a read-operation. *'in\_val'* is only needed in
case of a write operation and contains the byte written to the memory-mapped-IO
address. The return value is only needed for a read operation, in this case the
callback function needs to return a byte value which is handed back to the CPU.

The function pointer could either be stored in a new page structure member, but
we would better reuse the write\_ptr for this and set the read\_ptr to zero
(right now the read pointer must always be valid, so in the future a zero
read\_ptr would indicate that this page is used for memory-mapped-IO and
the write\_ptr is actually a function pointer).

Implementing memory-mapped-IO would slow down the memory system though, since
an additional check must be performed in each read and write operation.

#### What If: Wait States for the Z80 CPU

The easiest way to implement this would be through an internal counter which is
reset at the start of a CPU instruction and each memory access would increment
this counter by a number of cycles which would have to be associated with a
memory range when updating the memory mapping (so each page item would need an
additional member for the number of wait states an access to this memory page
incurs).

#### Less important things

The memory class has a few other methods which are less important but still
interesting to look at:

**Reading and writing 16-bit values**: this is useful for the Z80 CPU which has
a couple of instructions which read and write 16-bit values. Since the Z80 CPU
only has an 8-bit data bus, these are implemented though 2 separate memory
accesses. The memory class does exactly the same, but for a different reason.

16-bit memory accesses on the Z80 don't have to be aligned and might cross page
boundaries, or a wrap-around at address 0xFFFF might happen where the first
byte is read from address 0xFFFF and the second from 0x0000.

The memory class has the methods *r16()* and *w16()* for 16-bit reads/writes, which 
are implemented like this:

```cpp
inline uint16_t
memory::r16(uint16_t addr) const {
    uint8_t l = this->r8(addr);
    uint8_t h = this->r8(addr+1);
    uint16_t w = h << 8 | l;
    return w;
}

inline void
memory::w16(uint16_t addr, uint16_t w) const {
    this->w8(addr, w & 0xFF);
    this->w8(addr + 1, (w>>8));
}
```

As you can see, the Z80 is a little-endian CPU :) Another interesting tidbit
is that the wraparound from 0xFFFF to 0x0000 happens automatically, since the 
address parameter is a 16-bit unsigned value. This is one of the few cases where
C's automatic wraparound on overflow is actually useful.

**Reading signed byte values**: there is a minor convenience read-method which
returns the 8-bit value as a signed byte by simply casting the unsigned value
to signed before it is returned:

```cpp
inline int8_t
memory::rs8(uint16_t addr) const {
    return (int8_t) this->page_table[addr >> page::shift].read_ptr[addr & page::mask];
}
```

This is only used in the Z80 instruction decoder for relative jumps and indexed
addressing modes, where the 8-bit offset is interpreted as a signed number.

#### The End

And that's pretty much all there is to the memory system. The next blog post
will talk about the other 2 low-level components: the clock and the system bus!

