---
layout: post
title: "Zig and Emulators"
---

Some quick Zig feedback in the context of a new 8-bit emulator project I started
a little while ago:

[https://github.com/floooh/chipz](https://github.com/floooh/chipz])

Currently the project consists of:

- a cycle-stepped Z80 CPU emulator (similar to the emulator described
  here: https://floooh.github.io/2021/12/17/cycle-stepped-z80.html)
- chip emulators for Z80 PIO, Z80 CTC and an AY-3-8910 sound chip
- system emulators for Bombjack, Pengo and Pacman arcade machines,
  and the East German KC85/2../4 home computer series
- a code generation tool to create the Z80 instruction decoder code block
- various tests to check Z80 emulation correctness

With the exception of an external C dependency for 'host system glue'
(the cross-platform [sokol headers](https://github.com/floooh/sokol-zig) used for the window, input, rendering
and audio output), the project is around 16kloc of pure Zig code.

I'm not yet sure how this new project will evolve in relation to the [original
C/C++ 'chips' emulator project](https://github.com/floooh/chips), but I expect
that the Zig project will overtake the C/C++ project at some point in the
future.

## Dev Environment

I'm coding on an M1 Mac in VSCode with the [Zig Language Extension](https://marketplace.visualstudio.com/items?itemName=ziglang.vscode-zig), and [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)
for step-debugging.

The Zig and ZLS (Zig Language Server) installation is managed with ZVM (https://github.com/tristanisham/zvm).

For the most part this setup works pretty well, with a few tweaks:

- I'm doing 'build-on-save' to get more complete error information as described here:
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

Microchips of the 70s and 80s were very much like 'software libraries, but
implemented in hardware', they followed a minimal standard for interoperability
so that chips from different manufacturers could be combined into computer
systems without requiring too much custom glue between them. This 'competition
through interoperability' caused a cambrian explosion of cheap 8-bit
computer systems in the 70's and 80's.

Microchips communicate with the outside world via input/output pins, and a
typical 8-bit home computer system is essentially just a handful of microchips
communicating through their input/output pins.

The chipz project follows that same idea: The basic building blocks
are chip emulators which communicate with other chip emulators via
virtual input/output pins which are mapped to bits in an integer.

Chips of that era typically had up to 40 pins which makes them a good
fit for 64-bit integers used in today's CPUs.

The API of such a chip emulator only has one important function:

```zig
pub fn tick(pins: u64) u64
```

This tick function executes exactly one clock cycle, it takes an integer
as input where the bits represent input/output pins, and returns
that same integer with modified bits.

Fitting a CPU emulator into such a 'cycle-stepped model' can be a bit of
a challenge and is described in these blog posts (for the 6502 and Z80):

- [A new cycle-stepped 6502 CPU emulator](https://floooh.github.io/2019/12/13/cycle-stepped-6502.html)

- [A new cycle-stepped Z80 emulator](https://floooh.github.io/2021/12/17/cycle-stepped-z80.html)

A whole computer system is then emulated by writing a 'system tick function'
which emulates a single clock cycle for the whole system by calling the
tick functions of each chip emulator and passing pin-state integers
from one chips emulator to the next.

There's two related problems to solve with the above approach:

- There's not enough space to assign one bit for each inter-chip connection of
  a complete cokmputer system. This means a system tick function will
  need to maintain one pin-state integer for each chip, and shuffle bits
  around before each chip's tick function is called.
- For direct pin-to-pin connections it makes sense to assign the same bit position
  in different chip emulators, however, those direct connection are different
  in each emulated computer system. To make this work, a specialized chip
  emulator needs to be 'stamped out' for every computer system which assigns
  its pins to different bit positions in the pin-state integer.

Both problems can be solved quite elegantly in Zig:

- Instead of 64-bit integers for the pin-state we can switch to wide integers
  (u128, u192, u256, ...), which is enough room to assign each chip in a system
  its own reserved bit range instead of juggling with multiple 64-bit integers.
- With Zig's comptime generics it's possible to stamp out chip emulators
  which are specialized by a specific mapping of pins to bit positions in the
  shared wide integer.

This means a chip emulator is specialized by two comptime configuration values:

- a `Bus` type which is an unsigned integer with enough bits for all pin-to-pin
  connections in a system
- a `Pins` definition which defines a bit position for each input/output pin
  of a chip emulator

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

...which is used as nested struct in a `TypeConfig` struct which holds
all generic parameters:

```zig
pub const TypeConfig = struct {
    pins: Pins,
    bus: type,
};
```

This `TypeConfig` struct is used as parameter for a comptime Zig function
which returns a specialized type (this is how Zig does generics):

```zig
pub fn Type(comptime cfg: TypeConfig) type {
  return struct {
    // the returned struct is a new type which is configured
    // by the 'cfg' type configuration parameter
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

This specific `Z80` type uses a 128-bit pin-state integer and maps its own
pins to bit positions starting at bit 0, with the first 8 bits being the
data bus (most other chips in any computer system will also map their
data bus pins to the same bit range, since the data bus is usually shared
between all chips).

Note that `Z80` is just a type, not a runtime object. To get a default-initialized
Z80 CPU object:

```zig
var cpu = Z80{};
```

This example doesn't look like much, it's "just Zig code" after all, but this
is exactly what makes generic programming in Zig so elegant and powerful.

Arbitrarily complex comptime config options can be 'baked' into types,
and dynamic runtime configuration options can be passed in a 'construction' function
on that type, and all is just regular Zig code from top to bottom:

```zig
var obj = Type(.{
  // comptime options...
  .bus = u128,
  .pins = .{ ... },
}).init(.{
  // additional runtime options...
});
```

...and this is just scratching the surface. There's a couple of really
interesting side effects of this 2-step approach (first build the type,
then build an object from that type):

- Can use designated-init-syntax for configuring the type
  which is just **chef's kiss**.
- The TypeConfig struct can be composed by nesting other TypeConfig structs,
  or generic parameters in general, which then can be used to build
  types inside types (Yo Dawg...).
- It's possible to build different struct interiors based on comptime
  parameters (for instance the different KC85 models have different
  init-config structs, which makes 'misconfiguration' an immediate
  compile error).

In conclusion, the idea to use Zig's comptime features to stamp out specialized
per-system chip and system emulators works exceptionally well and is (IMHO)
*much* more enjoyable than C++ or Rust generic programming (I'm sure C++ and
Rust can do the same things with sufficient template magic, but this code
definitely won't look as nice as the Zig version).


## Bit Twiddling and Integer Math can be awkward

This section is hard to write because it's critizing without offering an
obvious solution, please read it as 'constructive criticism'. Hopefully Zig will
be able to fix some of those things on the road towards 1.0.

Zig's integer handling is quite different from C:

- arbitrary width integers are the norm, not the exception
- there is no concept of integer promotion in math expressions
- implicit conversion between different integer types is only
  allowed when no data loss can happen
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

The most extreme example I stumbled over was implementing the Z80's indexed
addressing mode (e.g. those instructions involving `(IX+d)` or `(IY+d)`. This
takes the byte `d` and adds it as a signed offset and with wraparound to a 16
bit address (e.g. the byte is sign-extended to a 16-bit value before the
addition).

In C this is straightforward:

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

```assembly
addi8:
  movsx eax, sil  ; move low byte of esi into eax with sign-extension
  add eax, edi    ; eax += edi
  ret
```

For ARM:
```assembly
addi8:
  ; looks like ARM handles the sign-extension right in the add instruction, not very RISC-y but neat!
  add w0, w0, w1, sxtb
  ret
```

IMHO when the assembly output of a compiler looks so much more straightforward
than the high level compiler input, it becomes a bit hard to justify why
high level programming languages had been invented in the first place ;)

Apart from the above extreme case (which only exists once in the whole code
base), narrowing conversions are much more common when writing code that
involved different integer widths, and those narrowing conversions require
explicit casts, and those explicit casts reduce readability.

IMHO the situation would much improve if the Zig compiler would just
be a but smarter.

The basic idea to only allow implicit conversions that can't lose data
is a good one, but very often a cast is required even though the
compiler has all the information it needs to prove that no
information is lost.

For instance this Zig code currently is an error:

```zig
fn trunc4(val: u8) u4 {
  return val & 0xF;
}
```

The expression result would fit into an u4, yet an `@intCast` or
`@truncate` is required to make it work:

```zig
fn trunc4(val: u8) u4 {
  return @intCast(val & 0xF);
}
```

Similar situation with a right-shift:

```zig
fn broken(val: u8) u4 {
  return val >> 4;
}

fn works(val: u8) u4 {
  return @truncate(val >> 4);
}
```

Somewhat surprisingly, this works fine:

```zig
  const a: u8 = 0xFF;
  const b: u4 = a & 0xF;
  const c: u4 = a >> 4;
```

A similar problem exists with loop variables, which are always of type usize and
which need to be explicitly narrowed even if the loop count is 'small enough':

```zig
for (0..16) |_i| {
  const i: u4 = @intCast(_i);
}
```

Without integer promotion, there's also surprising cases like this:

Assuming that:

  - a: u16 = 0xF000
  - b: u16 = 0x1000
  - c: u32 = 0x10000

This expression creates an overflow error:

```zig
  const d = a + b + c;
```

...but this doesn't:

```zig
  const e = c + a + b;
```

The type of `d` and `e` is both `u32` btw (which I find also a but surprising,
so Zig already seems to pick the widest input type as the result type, it 'just'
doesn't promote the other inputs to this widest type!

And here's another surprising behaviour I stumbled over:

```zig
// self.sprite_coords[] is an array of bytes
const px: usize = 272 - self.sprite_coords[sprite_index * 2 + 1];
```

This produces the error `error: type 'u8' cannot represent integer value '272'`.

The solution is to widen the value read from the array:

```zig
const px: usize = 272 - @as(usize, self.sprite_coords[sprite_index * 2 + 1]);
```

In conclusion, I only understood that C's integer promotion actually has an
important purpose after missing it so badly in Zig :D

I think C's main problem with integer promotion is that it promotes to `int`,
and int being stuck at 32-bits even on 64-bit CPUs (not moving the `int` type
to 64 bits during the transition from 32- to 64-bit CPUs was a pretty stupid
decision in hindsight).

TBF though, just extending to the natural word size (e.g. 64 bits) wouldn't
help much in Zig when using wide integers like u128. Maybe it makes sense to do
integer math on the width of the result type, or if that isn't available, the
width of the widest input.

In any case, I hope that the current status quo isn't what ends up in Zig 1.0
and that a way can be found to reduce '@-litter' in mixed-width integer expressions
without going back entirely to C's admittedly too sloppy integer promotion and
conversion rules.

Asking around on the Zig Discord there seems to be a proposal which lets
operators narrow the result type part of the expression is comptime
known (which would solve some of the problems mentioned above).

Another idea that might make sense is to add integer promotion to the
result type. Currently the compiler already seems to use the widest
input type in an expression as result type. Maybe it makes sense
to widen all other expression inputs to the result type.

I would keep the strict separation of signed and unsigned integer types
though, e.g. mixed signed expressions are not allowed, and any theoretical
integer promotion should happen 'across signedness'.

From my own experience in C (where I don't allow implicit signedness-conversion
via -Wsign-conversion warnings) I can say that this will feel painful
in the beginning for C and C++ coders, but it makes for better code and API
design in the long run.

This experience is also why I'm giving some slack to Zig's extreme integer
conversion strictness. After all, maybe I'm just not used to it yet. But OTH, I
have by now written enough Zig code that I should slowly get used to it, but it
*still* feels bumpy. All in all I think this is an area where 'strict design
purity' can the language in the long run though, and a balance must be found
between strictness, convenience and readability.


## Using wide integers with bit twiddling code is fast

Using a 128 bit integer variable for the emulator system bus works
nicely and doesn't have a relevant performance impact. In fact, with a bit of
care (by not using bit twiddling operations that cross a 64-bit boundary) the
produced assembly code is identical to doing the same operation on a simple
64-bit variable.

For instance extracting an 8-bit value from the upper half of an 128-bit integer:

```zig
fn getu8(val: u128) u8 {
  return @truncate(val >> 64);
}
```

...is just moving the register which holds the upper 64 bits into the return
value register:
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

Release performance of my C emulator code (with -O3) and my Zig code (with
-ReleaseFast) is roughly in the same ballpark, but I'm seeing a pretty big
difference in Debug performance:

- in C, debug performance is roughly 2x slower than -O3
- in Zig, debug performance is roughly 4x slower than ReleaseFast

I haven't figured out why yet, but it's not the most obvious candidate (range and
overflow checks) since ReleaseSafe performance is practically identical with ReleaseFast
(interestingly ReleaseSmall is the slowest Release build config, it's about 40% slower
than both ReleaseFast and ReleaseSmall).

One important difference between my C and Zig code is that in C I'm using tons
of small preprocessor macros to simplify expressions. In Zig these are replaced
with inline functions (`inline` in Zig isn't just an optimization hint, it causes
the function body to be inlined also in debug mode).

At first glance Zig's inline functions seem to be a good replacement for
C preprocessor macros, but when looking at the generated code in debug mode,
the compiler still pushes and pops function arguments through the stack even though
the function body is inlined.

Consider this Zig code:

```zig
inline fn add(a: u8, b: u8) u8 {
    return a +% b;
}

export fn add_1(a: u8, b: u8) u8 {
    return add(a, b);
}

export fn add_2(a: u8, b: u8) u8 {
    return a +% b;
}
```

...in release mode, both functions produce the same code as expected:

```assembly
add_1:
  lea eax, [rsi + rdi]
  ret

add_2:
  lea eax, [rsi + rdi]
  ret
```

But in debug mode, the function which calls the inline function has a slightly
higher overhead because of additional stack traffic:

```assembly
add_1:
  push rbp
  mov rbp, rsp
  sub rsp, 5
  mov cl, sil
  mov al, dil
  mov byte ptr [rbp - 4], al
  mov byte ptr [rbp - 3], cl
  mov byte ptr [rbp - 2], al
  mov byte ptr [rbp - 1], cl
  add al, cl
  mov byte ptr [rbp - 5], al
  mov al, byte ptr [rbp - 5]
  movzx eax, al
  add rsp, 5
  pop rbp
  ret

add_2:
  push rbp
  mov rbp, rsp
  sub rsp, 2
  mov cl, sil
  mov al, dil
  mov byte ptr [rbp - 2], al
  mov byte ptr [rbp - 1], cl
  add al, cl
  movzx eax, al
  add rsp, 2
  pop rbp
  ret
```

TBH though it's unlikely that inline function overhead is the only contributor to the
slower debug performance, but it could be many such small papercuts combined.

## Conclusion

TODO