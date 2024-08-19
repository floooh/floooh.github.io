---
layout: post
title: "Zig and Emulators"
---

> NOTE: I made several attempts for this blog post and it always ended up
> getting too big and rambly. As a result I'm omitting all background info
> about how the emulators actually work and will focus on Zig language
> features instead. A lot of this stuff either already exists in older blog posts, or
> should exists in other blog posts not yet written :)

Some quick Zig feedback in the context of a new 8-bit emulator project I started
a little while ago:

[https://github.com/floooh/chipz](https://github.com/floooh/chipz])

Currently this includes:

- a cycle-stepped Z80 CPU emulator (similar to the emulator described
  here: https://floooh.github.io/2021/12/17/cycle-stepped-z80.html)
- chip emulators for Z80 PIO, Z80 CTC and an AY-3-8910 sound chip
- system emulators for Bombjack, Pengo and Pacman arcade machines,
  and the East German KC85/2../4 home computer series
- a code generation tool to create the Z80 instruction decoder code block
- various tests to check Z80 emulation correctness

With the exception of an external C dependency for 'host system glue'
(the cross-platform sokol headers used for the window, input, rendering
and audio output), the project is pure Zig (unlike the original project
which is a mix of C, C++, Python for code generation and cmake scripts
for the build system).

Eventually I hope to bring the project on paar with, but this will take
a while (and next I'll go back to sokol-gfx feature development):

[https://github.com/floooh/chips](https://github.com/floooh/chips)


## Dev Environment

I'm coding on an M1 Mac in VSCode with the [Zig Language Extension](https://marketplace.visualstudio.com/items?itemName=ziglang.vscode-zig), and [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)
for step-debugging.

The Zig and ZLS (Zig Language Server) installation is managed with ZVM (https://github.com/tristanisham/zvm).

For the most part this setup works pretty well, with a few tweaks:

- I have setup 'build on save' to get more complete error information as described here:
  [Improving Your Zig Language Server Experience](https://kristoff.it/blog/improving-your-zls-experience/)
  (I'm not bothering with creating separate non-install build targets though)
- With the default Zig VSCode extension settings I was seeing that in long coding
  session (5..6 hours or so) saving would take longer and longer until it would
  eventually get stuck. After asking around on the Zig Discord this could be solved
  by explicitly setting the Zig Language Server as 'VSCode Formatting Provider'
  in the Zig Extension settings.
- When debugging, there's a somewhat annoying issue that the debug line information
  seems to be off in some places, the debugger appears to step into the last
  line of an inactive if-else block for instance. Again, Discord to the rescue,
  this seems to be a known regression.

All in all, not yet perfect, but good enough to get shit done.

## Zig Comptime and Generics

Before diving into language details, I'll need to provide some minimal
amount of background information of how the chipz emulators work:

Microchips of the 70s and 80s were very much like 'software libraries,
but implemented in hardware'. Microchips followed a minimal interoperability
standard so that chips from different manufacturers could be combined into
computer systems without requiring much 'custom glue', this basic idea of
competition through interoperability made the whole personal computer
revolution possible in the first place.

Microchips communicate with the outside world via their input/output
pins, and a complete computer system is essentially a handful of
microchips, connected through their pins.

The chipz project follows that same idea: The basic building blocks
are chip emulators which communicate with other chip emulators via
virtual input/output pins.

Chips of the home computer era typically had up to 40 pins, which
comfortably fits into a 64 bit integer.

The API of such a chip emulator only has one important function:

```zig
pub fn tick(pins: u64) u64;
```

This tick function executes exactly one clock cycle, it takes an integer
as input where the bits represent input/output pins, and returns
that same integer with modified bits.

Fitting a CPU emulator into such a 'cycle-stepped model' can be a bit of
a challenge and is described in these blog posts (for the 6502 and Z80):

- [A new cycle-stepped 6502 CPU emulator](https://floooh.github.io/2019/12/13/cycle-stepped-6502.html)

- [A new cycle-stepped Z80 emulator](https://floooh.github.io/2021/12/17/cycle-stepped-z80.html)

A system emulator for a whole computer system then 'simply' glues together a
handful such chips in a similar tick function which executes a single clock
cycle.

The main job of such of system tick function is to call the tick functions of
the chip emulators that computer is built from, plus some glue logic which is
called 'address decoding'.

In a traditional computer system, different subsystems (like the memory- and video-system)
and microchips are typically connected to a shared data bus. To avoid collisions
on this shared bus, a mediator process needs to be in place which decides what chip or
subsystem 'owns the bus' at any given clock cycle. This process is typically
called 'address decoding' but essentially comes down to 'if this specific
combination of chip output pins are active, activate this specific
combination of input pins of another chip'.

For instance a specific combination of CPU control- and
address-bus pins may activate the chip-select pin of a specific chip which
then knows that it is supposed to read or write the data bus.

To reiterate the above:

- virtual input/output pins mapped to bits in an integer
- chip emulators with a per-clock-cycle tick function
- system emulators as collection of connected chip emulators,
  also stepped forward with a per-clock-cycle tick function

With the above idea that each chip emulator uses a 64-bit integer to represent
its input/output pins at specific bit positions, we run into a couple of
awkward problems that need to be solved in a system tick function:

- while 64-bits is enough for the pins of a single microchip,
  it may not be enough to give each chip pin its own unique bit
  position in a whole system emulator
- if a computer system has multiple chips of the same type (for instance
  a C64 with its two CIA chips), the hardwired pin positions will collide
  with each other
- and finally, with hardwired pin positions we cannot directly express
  a connection between an output pin of one chip and an input pin
  of another chip, since those connections are computer system specific

In the [C version of the emulators](https://github.com/floooh/chips) these
problems are solved in the systemm tick function by maintaining a separate
pins-integer for each chip in the system, and bit shuffling code which moves
connected pins into the right position before and after ticking system chips.

In the Zig version I use a wide integer to represent the whole system
bus, and use Zig's comptime features to assign chip emulator pin
positions on this shared system bus.

For instance if an output pin of one chip is directly connected to an input
pin of another chip in a specific computer system, I can assign both
pins to the same bit position. This avoids the bit shuffling necessary
in the C emulators since the pin is already in the right bit position.

This is achieved by stamping out specialized chip emulators via Zig's
comptime and generics features:

Each chip emulator defines a struct which assigns pin names to bit positions.

For Z80 CPU emulator this pin definition struct looks like this:

```zig
pub const Pins = struct {
    DBUS: [8]comptime_int,
    ABUS: [16]comptime_int,
    M1: comptime_int,
    MREQ: comptime_int,
    IORQ: comptime_int,
    // ...
};
```
Apart from this pin mapping, the Z80 emulator also needs to know what integer
type to use for the system bus (u64, u128, etc...). All those comptime
parameters needed to 'stamp out' a specialized Z80 emulator are grouped
in a `TypeConfig` struct:

```zig
pub const TypeConfig = struct {
    pins: Pins,
    bus: type,
};
```

Next a comptime function is created which returns a specialized type (this
is how Zig does Generics):

```zig
pub fn Type(comptime cfg: TypeConfig) type {
  // define a type alias for our system bus type
  const Bus = cfg.bus;
  return struct {
    // define pin bit-masks from the pin positions
    pub const A0: Bus = 1 << cfg.pins.A[0];
    // ...
  };
}
```

...now we can stamp out a Z80 CPU emulator that's specialized for a specific
computer system by the system bus integer type and the Z80 pins mapped to
specific bit positions of this integer type:

```zig
const z80 = @import("z80");

const Z80 = z80.Type(.{
  .bus = u128,
  .pins = .{
    .DBUS = .{ 0, 1, 2, 3, 4, 5, 6, 7 },
    .ABUS = .{ 8, 9,  // ... },
    // ...
  }
});
```

Note that `Z80` is just a type, not a runtime object. To get a default-initialized
Z80 CPU object:

```zig
var cpu = Z80{};
```

This example doesn't look like much, it's "just Zig code" after all, but this
is exactly what makes generic programming in Zig so elegant and powerful.

Arbitrarily complex comptime configuration options can be 'baked' into types,
and dynamic runtime configuration options can be passed in a 'construction' function
on that type, and all is just regular Zig code from top to bottom:

```zig
var obj = Type(.{
  // comptime options...
}).init(.{
  // runtime options...
});
```

...and this is just scratching the surface. It's also possible to build
specialized nested types based on comptime type-config parameters of the parent
type. For instance the KC85/2, /3 and /4 models have different ROM
configurations and require different ROM dumps to be passed in:

```zig
pub fn Type(comptime model: Model) type {
  return struct {
    const Self = @This();

    pub const Options = struct {
      audio: Audio.Options,
      roms: switch (model) {
        .KC852 => struct {
          caos22: []const u8,
        },
        .KC853 => struct {
          caos31: []const u8,
          kcbasic: []const u8,
        },
        .KC854 => struct {
          caos42c: []const u8,
          caos42e: []const u8,
          kcbasic: []const u8,
        },
      },
    };

    // ...
    pub fn init(opts: Options) Self {
      ...
    }
  };
}
```

...note the `switch` statement in Options struct declaration, this causes a different
`Options` struct to be stamped out based on the `model` comptime parameter.

```zig
  // for KC85/2:
  pub const Options = struct {
    audio: Audio.Options,
    roms: struct {
      caos22: []const u8,
    },
  };

  // for KC85/3:
  pub const Options = struct {
    audio: Audio.Options,
    roms: struct {
      caos31: []const u8,
      kcbasic: []const u8,
    },
  };

  // for KC85/4:
  pub const Options = struct {
    audio: Audio.Options,
    roms: struct {
      caos42c: []const u8,
      caos42e: []const u8,
      kcbasic: []const u8,
    },
  };
```

...a KC85/3 emulator might then be created like this:

```zig
fn init() void {
  // ...
  sys = kc85.Type(.KC853).init(.{
    .audio = .{ // ... },
    .roms = .{
      .caos32 = @embedFile("roms/caos31.853"),
      .kcbasic = @embedFile("roms/basic_c0.853"),
    },
  });
  // ...
}
```

All in all, the idea to use Zig's comptime features to stamp out specialized
per-system chip emulators, which can then be connected to each other with less
runtime glue code worked very well. While overall performance
doesn't seem to be any better or worse than the C version (with -O3 vs ReleaseFast), the simplified
system tick function and reduced runtime state is worth it.


## Bit Twiddling and Integer Math Woes

This section is hard to write because it's critizing without offering an
obvious solution, please read it as 'constructive criticism'. Hopefully Zig will
be able to fix some of those things on the road towards 1.0.

Zig's integer handling is quite different from C:

- arbitrary width integers are the norm, not the exception
- there is no integer promotion which converts items in math expression to a
  common type
- assignment between different integer types is only allowed if no data loss
  can happen
- mixing signed and unsigned values in expressions isn't allowed
- overflow is checked in Debug and ReleaseSafe mode, but there are separate
  operators for wraparound

At first glance these features look pretty nice because they fix some obvious
footguns in C and C++. Arbitrary width integer types are especially useful for
emulator code, because hardware chips are full of 'odd-width' counters and
registers (3, 5, 20 bits etc...). Directly mapping such registers to types like
u3, u5 or u20 should potentially allow for more readable and 'expressive' code.

Unfortunately, in reality it's not so clear cut. While C is definitely too
sloppy when it comes to integer math, Zig might swing the pendulum a bit too
far into the other direction by requiring too much explicit casting.

The most awkward example I stumbled over is the indexed addressing mode of the
Z80 (e.g. instructions involving `(IX+d)` or `(IY+d)`). This takes the byte `d`
and adds it as a signed offset and with wraparound to a 16 bit address (e.g.
the byte is sign-extended to a 16-bit value before the addition).

In C this is expressed as:

```c
uint16_t addi8(uint16_t addr, uint8_t offset) {
  return addr + (int8_t)offset;
}
```

The simplest way I could come up with to do the same in Zig is:

```zig
export fn addi8(addr: u16, offset: u8) u16 {
  return addr +% @as(u16, @bitCast(@as(i16, @as(i8, @bitCast(offset)))));
}
```

Note how the intent gets totally drowned in '@-litter'.

Both functions result in the same x86 and ARM assembly output (with -O3 for C
and any of the Release modes in Zig):

```
addi8:
  movsx eax, sil  ; move low byte of esi into eax with sign-extension
  add eax, edi    ; eax += edi
  ret
```

For ARM:
```
addi8:
  ; looks like ARM handles the sign-extension right in the add instruction, not very RISC-y but neat!
  add w0, w0, w1, sxtb
  ret
```

IMHO when the assembly output of a compiler looks much more straightforward
than the high level language input, it's time to stop and check if we're still
on the right path ;)

Apart from the above extreme case (which only exists once in the whole code
base), narrowing conversions are much more common to deal with, either via
`@truncate()` (which simply cuts off the top bits), or `@intCast()` (which does
a runtime check in Debug and ReleaseSafe mode).

There are situations where the compiler should know that a narrowing cannot
lose data, but where a `@truncate()` or `@intCast()` is still required, for
instance:

```zig
fn trunc4(val: u8) u4 {
  return @truncate(val & 0xF);
}
```

...since the `& 0xF` already explicitly cuts off the top bits, the `@truncate()`
should be redundant but is currently required.

...but somewhat surprisingly, this works:

```zig
  const a: u8 = 0xFF;
  const b: u4 = a & 0xF;
```

A similar problem with loop variables, which are always of type usize and
which need to be explicitly narrowed even if the loop count is 'small enough':

```zig
for (0..16) |_i| {
  const i: u4 = @intCast(_i);
}
```

...the compiler *should* know that the loop variable fits into an u4 without
data loss, but currently still requires an explicit cast.

Without integer promotion, there's also surprising cases like this:

```zig
  const a: u16 = 0xF000;
  const b: u16 = 0x1000;
  const c: u32 = 0x10000;

  // this produces an overflow error:
  const d = a + b + c;

  // but this doesn't:
  const e = c + a + b;
```

Then there bit shifting. The shift amount is a narrow integer type which can
hold the maximum shift value to prevent 'overshifting'. For instance in
the expression:

```zig
  const c = a << b;
```

If `a` is of type `u8`, then `b` is expected to be of type `u3`. When using a
comptime-known shift amount this creates a useful error if the shift-value
is out-of-bounds:

```zig
  const a: u8 = 255;
  const c = a << 8;   // => error: type 'u3' cannot represent integer value '8'
```

...but it gets ugly if the shift amount of a wider type, for instance
when it is coming out of a loop):

```zig
  const a: u8 = 255;
  for (0..7) |i| {
    const c = a << i; // error: expected type 'u3', found 'usize'
  }
```

...so this requires an explicit `@intCast()` or `@truncate()`, even though the
compiler should be able to figure out that shift value fits into an u3:

```zig
  const a: u8 = 255;
  for (0..7) |i| {
    const c = a << @intCast(i);
  }
```

...or forcing the loop to be unrolled at comptime, because then the loop variable
is a more flexible `comptime_int`:

```zig
  const a: u8 = 255;
  inline for (0..7) |i| {
    const c = a << i;
  }
```

...here's a bit of a footgun:

```zig
fn shift(val: u8, shift: u3) u16 {
  return val << shift;
}
```

When called with `shift(0xFF, 7)` this function returns 0x80 and not 0x7F80 since
the top bits are shifted out of the u8 range before the result is extended to
u16.

The correct form involves adding `@-litter` again, the 8-bit values needs to be
manually extended to 16-bits before the shift operation:

```zig
fn shift(val: u8, shift: u3) u16 {
  return @as(u16, val) << shift;
}
```

And here's another surprising behaviour I stumbled over. `sprite_coords` is an u8-array:

```zig
const px: usize = 272 - self.sprite_coords[sprite_index * 2 + 1];
```

This produces the error `error: type 'u8' cannot represent integer value '272'`.

The solution is to widen the value read from the array:

```zig
const px: usize = 272 - @as(usize, self.sprite_coords[sprite_index * 2 + 1]);
```

TL;DR: I only understood how helpful C's integer promotion actually is
after not having it available in Zig :D

I think C's main problem is that it promotes to `int`, and int being stuck at
32-bits even on 64-bit CPUs (in hindsight an exceptionally stupid decision).

TBF though, just extending to the natural word size (e.g. 64 bits) wouldn't
help much in Zig when using wide integers like u128. Maybe it makes sense to do
integer math on the width of the result type, or if that is inferred, on the
width of the widest input.

In any case, I hope that the current status quo isn't what ends up in Zig 1.0
and that a way can be found to reduce '@-litter' in mixed-width integer expressions
without going back entirely to C's admittedly too sloppy integer promotion and
conversion rules.


## Using wide integers in bit twiddling is fine

Using a wide integer (like u128, u256 etc...) for the system bus works nicely
and doesn't have the negative performance impact I feared. In fact, with a bit
of care (by not crossing 64-bit boundaries) the produced assembly code is
identical with the same operation on a 64-bit variable.

For instance extracting an 8-bit value from the upper half of an 128-bit integer:

```zig
fn getu8(val: u128) u8 {
  return @truncate(val >> 64);
}
```

...is just a move from one register into another:
```
getu8:
  mov rax, rsi
  ret
```

...which is the same cost as extracting an 8-bit value from a 64-bit variable:

```zig
fn getu8(val: u64) u8 {
  @truncate(val);
}
```

```
getu8:
  mov rax, rdi
  ret
```

...just make sure that the operation doesn't cross 64-bit boundaries:

```zig
fn getu8(val: u128) u8 {
  return @truncate(val >> 60);
}
```

...because this now involves actual bit twiddling:
```
getu8:
  shl esi, 4
  shr rdi, 60
  lea eax, [rdi + rsi]
  ret
```

## Debug Performance

## Strings, Slices and Memory
