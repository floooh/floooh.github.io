---
layout: post
title: First Steps in Rust
---

**TL;DR:** some very early noob impressions after a few days of Rust from
the perspective of a C, C++ and Python coder.

**NOTE:** I think I should make clear first that I'm very excited about Rust,
even though some of the points in this post sound not _that_ enthusiastic :) In
the bigger picture, these are just minor nuisances, some are probably just my
brain being tuned too much to C/C++, others will hopefully get fixed in time.

### Alright, Start:

I finally decided to spend some time with Rust. In the past I've been reading a
bit in the Rust Book, did a few tutorials, but never got further than midway
through the Dining Philosophers sample. These coding tutorials don't work for me
to learn a language, it just doesn't stick. I need to work on my own code, and
if I'm stuck on a problem actively find a solution (mostly through googling and
looking up the documentation, but still, somehow this works better for me than
working through pre-baked tutorial code).

So here's what I'm doing: a simple Z80 emulator in Rust, which
may grow into a full 8-bit home computer emulator later.

Github project is here: [https://github.com/floooh/rz80](https://github.com/floooh/rz80).

### Why a Z80 emulator:

- having written a C++ home computer emulator in the past few months, it's all
  still fresh in my head:
  [http://www.github.com/floooh/yakc](http://www.github.com/floooh/yakc)
- a CPU emulator is self contained, 'pure' code. It doesn't have to call into
  the operating system or external libraries, there's no dynamic memory
  management at all.
- without dynamic memory issues, I can first focus on the simple parts of Rust
  before having to deep-dive into all the ownership and borrow-checker stuff.
- ...and finally, from the C++ implementation I have a good idea what
  performance to expect and can compare the two.

### What I'm looking for in Rust:

Sanity and a quiet retreat from the 'Modern C++ Circus', at least for some
time. 

For performance-sensitive game-development-related things, the 'orthodox 
subset' of C++ is as good as it gets, C++14 and 17 don't improve the language
in any of the areas I'm interested in.

In Rust, I'm not interested in the high-level 'functional style' 
like this:

```rust
    // Functional approach
    let sum_of_squared_odd_numbers: u32 =
        (0..).map(|n| n * n)             // All natural numbers squared
             .take_while(|&n| n < upper) // Below upper limit
             .filter(|n| is_odd(*n))     // That are odd
             .fold(0, |sum, i| sum + i); // Sum them
    println!("functional style: {}", sum_of_squared_odd_numbers);
```

Call me a traditionalist simpleton, but the following code makes much more
sense (and is most likely much easier to debug):

```rust
    let mut acc = 0;
    for n in 0.. {
        let n_squared = n * n;
        if n_squared >= upper {
            break;
        } else if is_odd(n_squared) {
            acc += n_squared;
        }
    }
    println!("imperative style: {}", acc);
```

What I'm looking for in Rust is a type-safe, memory-safe,
slightly higher-level C without any hidden runtime magic for
memory management.

I still want to be able to glimpse the assembly code between the
lines, in the sense that I want to be able to estimate whether a line of high
level code translates to 1, 10 or 100 assembly instructions, and what the
memory access patterns roughly look like.

It may well be that the Rust world has (or will have) Functional Zealots in the
same way that the C++ world has its Modern C++ Zealots, and that they will move
the language into the 'wrong direction' (from my point of view), but for now I
don't care :)

### To the Batmobile! Let's go:

Ok, back to the topic at hand. What follows is an unordered list of things I
stumbled over, found interesting, where I scratched my head or got stuck, all
from a complete Rust noob, so take everything with a grain of salt.

**Build System / Dependency Management (cargo)**: this is easily the best part
of the whole Rust ecosystem, not because cargo is particularly awesome, but
because it is **the only option** to choose from. There's so much energy lost
in the C++ world to decide between cmake, premake, WAF and whatnot (not to
mention endless discussions and pointless arguing what's the best or most
standard build system) that not much energy is left to actually start writing
any code. It's bizarre.

**Edit/Compile/Test**: I'm using vim (or rather NeoVim) for my Rust
experiments. For C/C++ I usually work in Xcode or VStudio (not because I think
IDEs are particularly useful, but because the debugger is only one key-press
away).  For Rust I've installed 2 vim plugins: _scrooloose/syntastic_ and
_rust-lang/rust.vim_. Together they provide syntax highlighting and some loose
syntax checking (like missing braces). I compile and test directly from vim
with **:!cargo build** and **:!cargo test**. What I'm missing is a plugin which
parses the compile errors and highlights them in vim (for C/C++, the plugin
_m21/errormarker.vim_ does this, not for Rust error messages unfortunately).

**Debugging**: with this I struggled a few days, but that's mostly OSX 10.12's
fault.  Rust creates DWARF debugging info for gdb and lldb, and while I feel
more at home in a proper IDE debugger like Visual Studio's I also don't scoff
at cgdb. But that's where my tolerance ends, working in 'raw' gdb or lldb is
definitely not a lot of fun. Doesn't matter though since gdb doesn't work in
OSX 10.12 anyway, at least I didn't get it to work after much frustrating
dabbling. For some brain-dead security reasons gdb needs to be code-signed in
OSX, and I spent a few hours trying to get this to work, but finally gave up. I
suspect that Apple changed something code-signing-related in OSX 10.12 (again)
since I'm pretty sure I had cgdb working before.

So the lldb debugger coming with the Xcode command line tools it is then. After
a few more hours of googling around and trying things out I finally found a
solution that works pretty well, even though it is only a very simple one:

**Visual Studio Code with the [LLDB Debugger extension](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)**:

![vscode-debug](/images/vscode-dbg.png)

This allows to set breakpoints, step through the code and inspect local
variables (that's all I've tried so far). One feature I've been missing
immediately is to show variables as hex (kinda important for a CPU emulator).

**Noisy Type Casting**: C doesn't require explicit casting between integer
types (e.g. uint8\_t to uint16\_t or unsigned to signed). I always thought that
C would be a better language if it didn't do automatic type casting, but after
seeing this in action in Rust I'm not so sure anymore.  Initially I used 8-bit
and 16-bit unsigned integer types for the CPU registers, but this resulted in a
manual type cast in nearly each line of code, and compared to the original C++
code this looked really noisy. I worked around this for now by using a wider,
common integer type (currently i64) everywhere in the CPU structure. This
avoids all the typecasting, but I need to handle overflow manually by adding a
**& 0xFF** or **& 0xFFFF** in many places (and if I forget this in one place, I
have a potentially hard to find bug in the emulation). Speaking of overflow:

**Integer Overflow is a Runtime Error**: I can understand why Rust does this
but boy is it annoying in the context of an 8-bit CPU emulator, since this
requires overflow on 8- and 16-bit types all the time. Rust has a special set
of overflow-wrapping functions, you can write this for instance:

```rust
    let x = 0xFFu8.wrapping_add(1);
```

But at least to me this looks a lot messier than just:

```c
    uint8_t x = 0xFF + 1;
```

**No Unions**: Not a big deal, but I'm doing this neat trick in C++ to
access the Z80 8-bit / 16-bit register pairs:

```cpp
    union {
        struct { uint8_t C,B; };
        uint16_t BC;
    };

```

I haven't found anything similar in Rust, so I have to build BC 'by hand'
by shifting and or-ing B and C together.

**if is an Expression**: ok, a nice thing for a change :)

In Rust you can write:

```rust
    let x = if y == 1 { 2 } else { 3 };
```

At first glance this looks worse than C's ternary operator:

```c
    int x = (y == 1) ? 2 : 3;
```

But in Rust it's always the full-monty if-else, not some specialized
language construct.

**The last 'dangling' expression is the return value**: This may look nice in
some cases (for instance in the if-else above), but totally silly in others.
For instance I have a method which implements the Z80 DJNZ instruction and
returns the number of cycles taken:

```rust
    pub fn djnz(&mut self) -> i32 {
        self.reg[B] = (self.reg[B] - 1) & 0xFF;
        if self.reg[B] > 0 {
            let d = self.mem.rs8(self.pc);
            self.wz = (self.pc + d + 1) & 0xFFFF;
            self.pc = self.wz;
            13 
        }
        else {
            self.pc = (self.pc + 1) & 0xFFFF;
            8 
        }
    }
```

That dangling 13 and 8 at the end of the if-else branches takes a lot 
of getting used to IMHO (you can write 'return 13;' but apparently that's
considered 'un-rustic').

**Testing**: Now this is really nice, especially when
writing a CPU emulator, where I'm writing a lot of small
instruction-level tests in parallel to the implementation. For instance
here's a small test function which (very roughly) tests the implementation
of the Z80 AND instruction. I've put this directly into the CPU implementation
file:

```rust
    #[test]
    fn and8() {
        let mut cpu = cpu::new();
        cpu.reg[A] = 0xff; cpu.and8(0x01);
        assert!(0x01 == cpu.reg[A]); assert!(test_flags(&cpu, HF));
        cpu.reg[A] = 0xff; cpu.and8(0xaa);
        assert!(0xaa == cpu.reg[A]); assert!(test_flags(&cpu, SF|HF|PF));
        cpu.reg[A] = 0xff; cpu.and8(0x03);
        assert!(0x03 == cpu.reg[A]); assert!(test_flags(&cpu, HF|PF));
    }
```

...and in a separate integration test module (also in a standard location)
I have a number of higher level tests which run the entire instruction
decoding- and execution loop, for instance this little Z80 program
to test the DJNZ instruction:

```rust
    #[test]
    fn test_djnz() {
        let mut cpu = rz80::CPU::new();
        let prog = [
            0x06, 0x03,     // LD B,0x03
            0x97,           // SUB A
            0x3C,           // loop: INC A
            0x10, 0xFD,     // DJNZ loop
            0x00,           // NOP
        ];
        cpu.mem.write(0x0204, &prog);
        cpu.pc = 0x0204;

        assert!(7  == cpu.step()); assert!(0x03 == cpu.reg[rz80::B]);
        assert!(4  == cpu.step()); assert!(0x00 == cpu.reg[rz80::A]);
        assert!(4  == cpu.step()); assert!(0x01 == cpu.reg[rz80::A]);
        assert!(13 == cpu.step()); assert!(0x02 == cpu.reg[rz80::B]); assert!(0x0207 == cpu.pc);
        assert!(4  == cpu.step()); assert!(0x02 == cpu.reg[rz80::A]);
        assert!(13 == cpu.step()); assert!(0x01 == cpu.reg[rz80::B]); assert!(0x0207 == cpu.pc);
        assert!(4  == cpu.step()); assert!(0x03 == cpu.reg[rz80::A]);
        assert!(8  == cpu.step()); assert!(0x00 == cpu.reg[rz80::B]); assert!(0x020A == cpu.pc);
    }   
```

Nearly after each single line writing those tests, I do a **:!cargo test**
from inside vim, and get the result in a fraction of a second:

![rust-test](/images/rust-test.png)

Integration with TravisCI is also extremely simple. Just add the following
.travis.yml file to the project root:

```yaml
language: rust
rust:
    - stable
    - beta
    - nightly
matrix:
    allow_failures:
        - rust: nightly
```

This builds the project against 3 Rust versions (stable, beta and nightly), runs
the tests and produces the following [output (click
me)](https://travis-ci.org/floooh/rz80/jobs/140595512).

**One slightly annoying borrow checker bug(?)**: I stumbled over one very simple
but annoying problem where the borrow checker seems to give a false-positive warning.

A similar problem is detailed here (I had that one too):
[https://github.com/rust-lang/rust/issues/29975](https://github.com/rust-lang/rust/issues/29975)

Here's the problem in a nutshell (including my possibly wrong interpretation):

A couple of methods to read and write emulator memory, and decrement
a value:

```rust
    /// read a byte from memory
    pub fn r8(&self, addr: u16) -> u8 {
        // ...
    }
    /// write a byte to memory
    pub fn w8(&mut self, addr: u16, val: u8) {
        // ...
    }
    /// decrement value and update CPU flags
    pub fn dec8(&mut self, val: u8) -> u8 {
        // ...
    }
```

In the actual instruction decoder method I want to implement the
Z80 instruction DEC (HL), which loads a byte from memory at address HL,
decrements it, and writes the result back to the address at HL.

I **should** be able to write this as a one-liner like this:

```rust
    pub fn decode(&mut self) {
        // ...
        self.w8(addr, self.dec8(self.r8(addr));
        // ...
    }
```

This first calls r8(), then dec8(), then w8() all sequentially one after
the other. But Rust complains that it 'cannot borrow *self as mutable
more than once at a time'. May be I'm wrong, but I think that's a false
positive, the following code should be completely equivalent and is accepted:

```rust
    pub fn decode(&mut self) {
        // ...
        let x = self.r8(addr);
        let y = self.dec8(x);
        self.w8(addr, y);
        // ...
```

Ok, enough for today. I hope to do a couple more blog posts like this as
I progress with the emulator. One interesting problem I'll need to solve is 
for instance: how do I get a Z80 program dump from a binary file converted
into a Rust array inside a source file. In C/C++ I use python code generation
for this, fully integrated into the build process. Not sure yet what's the 
best way in Rust.


