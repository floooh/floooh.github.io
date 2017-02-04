---
layout: post
title: "YAKC Emu: Counters Everywhere"
---
**TL;DR:** about clock ticks and counters

When I started with the [YAKC Emulator](http://floooh.github.com/virtualkc)
about a year ago, home computers were essentially still magic boxes to me -
even though I was programming simple games on those 8-bit machines a long time
ago.

They produce pretty pictures and sound, and when you poke them in the right
places, the pictures and sounds change (that's still essentially how game
programming works, or at least how it _should_ work), but it isn't immediately
obvious how an action (poking it) leads to a result (new colors and sounds). So
basically: magic!

But the thick spellbooks that came with 80's home computers tried their best
to lure the unsuspecting reader into the dark arts by teaching them BASIC, a
long forgotten demonic language created to mess up the minds of innocent kids.
Many jumped into the rabbit hole and never found their way back into a normal
life. Some even ended up as game programmers!

Today everything is much better of course. Here is your shiny new device, don't
look inside, don't ask how it works that's really not that important, we just
want you to feel safe and comfortable. Now, would you kindly sign this 50-page
License Agreement with your blood? Just kidding, it's just a simple button
press...

### Peeling the magic onion

Ok, how does that relate to emulator timing? Not at all of course :D

Except: this process of peeking under that layer of magic when uttering the
first words of BASIC repeats several times when diving into emulator coding,
and what lies at the bottom is the essence of how to get a really accurate
emulator.

But one step at a time.

In the beginning you go by the home computer's programming manual, poke a value
here: sound frequency changes! Poke a value there: a pixel appears! So you
treat the programming manual like a cook book, but don't pay much attention how
it all really works under the hood. With this high-level approach, the only
complicated part is the CPU emulation, because this has not much room for
skipping the details. But once the CPU works the rest isn't that complicated,
write some code that decodes the video memory bytes for a whole frame into a
flat RGBA8 texture, write key presses into the right memory location, and for
most home computer systems this is enough to boot up and play around a bit. 

This is in fact how I usually start a new emulated system. Find out
where the video memory is and how it is layed out, find out what's the cheapest
way to get keyboard input into the system, and then do some quick'n'dirty hacks
just to get the operating system boot up and have it accept key presses.

After that a deeper exploration phase starts. Dig into the technical
documentation (which thankfully had been scanned before they rotted away, and
preserved through the 1990's Dark Age by grey-bearded monks living in
remote mountain abbeys). Then with this more detailed information
improve the emulation step by step.

After a while you begin to notice that home computers were built mostly
from a small number of different chip types, especially the Eastern European 
models. 


### Integrated Circuits

Integrated Circuits (ICs) are the hardware counterpart of software libraries.
They have a public API (pins sticking out at the sides) and a documentation
that explains how the input pins must be poked to get some useful result at the
output pins. The documentation is usually available as a completely illegible
PDF scan, and only if you're lucky the scan has been run through OCR so that 
text search is possible. The mountain abbey responsible for the IC datasheets
must have been a bit understaffed.

From the outside, ICs are also just smaller magic boxes: poke it here, something twitches 
over there. If you're just writing an emulator for a single system you can
leave it at that. It's enough to implement just the outside behaviour of the 
magic box for this specific system.

But for multi-system emulators it really pays off to peel the next onion skin
layer and look inside the box, because once an IC's internal behaviour is
(somewhat) accuratly implemented in its own 'software module' you can plug that
module into the emulator for another system. And once you have the most
common chip types in your collection, supporting a new system is then
mostly a game of connect-the-dots (or rather connect-the-pins).

Adding new emulated systems becomes an addicting meta-game: you start looking
for systems that share common chip types, and once one of those common chips
has been implemented it paves the way toward another emulated system!

There was an interesting difference between Western and Eastern 8-bit homecomputers
in how they used ICs though:

Western home computers often were designed around one big custom chip (usually called
the 'Gate Array' or 'ULA') to **reduce production costs**.

Eastern home computers were often designed around a handful of off-the-shelf chips to 
**reduce production costs**.

_Wat_? 

For the high production volumes of the popular Western models it seems to have
been cheaper to integrate as many functions as possible into a single chip,
since at high volume, building one custom chip is cheaper than buying 3 or 4
off-the-shelf chips even if the development cost must be amortized and production
capacity rented. Some home computers also started with a bunch of standard ICs
which moved gradually into the gate array in later hardware revisions.

In the East, the 8-bit 'home-computer-style' systems only played second fiddle
to office computers. The home computers had to use the 'reject chips' that
didn't pass quality control and couldn't run at full speed. Chip production
capacity was just enough to get the office computer production going. There was
simply no way a hardware engineer could go to his boss and propose a custom
chip design, he would have been laughed out the door. Buying parts in the West
was also not an option, this would have required 'hard currency', and apart
from that, there was the CoCom embargo, Western companies were forbidden by
their governments to sell technology into the East that was more advanced than
a millstone.

The only interesting exception I know of was the sound chip in the KC Compact,
which was a late East German CPC clone.  While the necessary CPC custom gate
array functions were emulated using available standard chips (a solution which
actually offered a bit more programmability than the original gate array), the
KC Compact sound chip (an AY8912) was a Taiwanese import. But that was in 1989
when 8-bit tech was hardly relevant any more in the West.

Ok, back to emulators.

After I added the first systems with dedicated video- and audio-chips to my
emulator (the first one was the Amstrad CPC, and more recently Acorn Atom) I
started to recognize what's under the next magic onion layer:

### Oh my dog, it's full of counters!

Those video- and audio-chips are essentially just a bunch of counter cascades! A
counter counts to a specific value, when the value is reached it 'ticks'
another counter forward and then starts counting at 0 again. All those
complicated, magic ICs are more or less just medieval clockworks inside!

And after that first realization, those counters are suddenly everywhere,
with the exception of the CPU and keyboard matrix, a home computer is essentially
just a lot of counters that tick other counters.

I'll illustrate the point with a little GEDANKENEXPERIMENT!

Let's build a little idealized home computer, without a CPU, keyboard or sound, so 
not very useful. The theoretical computer should just display an image on
a TV.

Instead of supporting the PAL or NTSC video standard, we'll invent a highly
simplified video standard, this is fine since it's just a GEDANKENEXPERIMENT!

The invented video standard has 3 binary input signals: COLOR, HSYNC and VSYNC.

A video beam travels over the screen along a horizontal line until the HSYNC
signal flips to ON for one tick. When this happens, the video beam jumps back
to the left edge of the screen and one line below, and repeats the whole
process for the next line.

The COLOR signal can be switched ON or OFF at any time.  When it is ON, the
video beam will 'light up' and a white pixel is rendered, when the COLOR signal
is OFF, the video beam will go dark and black is rendered.

When the VSYNC signal switches to ON for one tick, the video beam shall jump
back to the top left corner of the screen, and the whole procedure starts again.

Also let's just assume that the 'retrace' when the video beam jumps back to the
beginning of the next line, or back to the top-left corner happens instantly.

Note that the invented video standard doesn't say anything about _when_ exactly HSYNC
and VSYNC is triggered or how often the COLOR signal can go on or off, here's 
how it would look like for a 16x8 pixel display, rendering a 'plus':

```
    >--------------->    '-': COLOR OFF
    >--------------->    'X': COLOR ON
    >------XX------->
    >------XX------->
    >---XXXXXXXX---->
    >------XX------->
    >------XX------->
    >---------------> < VSYNC
                    ^
                    HSYNC
```

Let's build a simple video chip which can generate an image like this.

The chip needs to feed our imaginary video standard, so it needs 3 **output
pins** for the 3 video signals:

- **bool COLOR**:    OFF to render black, ON to render white
- **bool HSYNC**:    toggles to ON for 1 clock cycle to trigger the horizontal retrace
- **bool VSYNC**:    toggles to ON for 1 clock cycle to trigger the vertical retrace

...and 1 **input pin** with the 'clock tick'. The frequency of this clock tick
is the same as the 'pixel output frequency' of our video system.

We also need a few counters to keep track of when to trigger the HSYNC and VSYNC
signals, and where to fetch video memory bytes from:

- **int HCOUNT**:   counts from 0 to 16
- **int VCOUNT**:   counts from 0 to 8
- **int ADDR**:     counts up and is reset when VSYNC occurs

First let's take care of the COLOR output pin. This should go ON when the video 
memory byte at the ADDR counter is greater 0, otherwise OFF (apologies for abusing
the ADDR counter as a pointer in the following pseudocode):

```
void on_tick() {
    COLOR = *ADDR > 0;
}
```

There's not much to see yet though, since our video beam will have wandered off into
the void beyond the right edge of the screen... Let's take care of the horizontal 
retrace. The HCOUNT counter is directly connected to the input clock and keeps track
of the horizontal video beam position. To prevent the video beam from wandering beyond
the right screen edge the HSYNC pin must be triggered and the HCOUNT needs to start at
zero again:

```
void on_tick() {
    COLOR = *ADDR > 0;
    HCOUNT++;
    if (HCOUNT == 16) {
        HSYNC = true;
        HCOUNT = 0;
    }
    else {
        HSYNC = false;
    }
}
```

Ok, our video signal generator chip will now switch the HSYNC pin to ON for 1 tick which 
causes the video beam to perform a 'horizontal retrace' (it jumps back to the left edge).
We must take care to switch HSYNC off for the rest of the line, otherwise the beam would 
immediately perform a horizontal retrace after the first pixel. 

All good and well, 1 frame will be rendered, but then the video beam will wander off into
the void below the screen. We need to generate a short VSYNC signal to force the beam 
back to the top-left corner. This works the same way as the HSYNC, except that the
vertical counter is only bumped once per scanline. When the VCOUNT reaches 8, VSYNC
will be triggered to ON for 1 tick (and all other ticks it will be forced to OFF).

HCOUNT and VCOUNT now form a counter cascade!

```
void on_tick() {
    COLOR = *ADDR > 0;
    HCOUNT++;
    if (HCOUNT == 16) {
        HSYNC = true;
        HCOUNT = 0;
        VCOUNT++;
        if (VCOUNT == 8) {
            VSYNC = true;
            VCOUNT = 0;
        }
    }
    else {
        HSYNC = false;
        VSYNC = false;
    }
}
```

Ok, this takes care of the video beam control. The beam is caught and brought back at 
the end of a scanline, and at the end of the screen.

Let's go back to the COLOR pin. So far this will be either ON or OFF for the
whole screen, depending on what's in the random byte pointed to by ADDR. The
ADDR counter isn't incremented yet, so the whole screen will be either
completely black (if *ADDR == 0) or completely white (if *ADDR != 0). Let's
bump the ADDR counter up each tick so that we read out a new video memory byte
each tick. Also, we need to reset the ADDR counter at the end of the screen
so that it starts scanning at the beginning of video memory again.

```
void on_tick() {
    COLOR = *ADDR > 0;
    ADDR++;
    HCOUNT++;
    if (HCOUNT == 16) {
        HSYNC = true;
        HCOUNT = 0;
        VCOUNT++;
        if (VCOUNT == 8) {
            VSYNC = true;
            VCOUNT = 0;
            ADDR = 0;
        }
    }
    else {
        HSYNC = false;
        VSYNC = false;
    }
}
```

And that's it! This is basically how a simple video signal generator works (for
instance the Motorola MC6845 and MC6847 chips), both in the real world and in
cycle-accurate emulators.

In reality there are a few annoying details, like that the retrace doesn't
happen instantly, the details are described in the PAL and NTSC specification
and video chip datasheets, but all those additional real world details don't
change the basic idea of cascaded counters, they just add a bunch more counters.

Counter cascades are also used in other places:

- **audio chips**: These work much like the video signal generator above but at
  a much lower frequency, a 'period counter' would count up to a specific
  value, and when the counter hits its limit a flip-flop bit will be toggled.
  When the bit is ON, the speaker membrane on the other end will push outward,
  creating a little wave of compressed air hitting our ear. When this happens a
  few hundred- to a few thousand times per second, we can hear a sound. All
  home computer sound generators are based on that simple idea, the most simple
  version is just a CPU-driven square-wave-beeper, and the more advanced audio
  chips have multiple audio channels, different wave forms and envelope
  generators. But at the core these are all built with cascaded counters (well, at
  least those I encountered so far).

- **counter/timer chips**: A number of ICs offered counting and timing
  functions (for instance the Z80-family CTC or the MOS 6522), this was often
  used to communicate with external devices, like tape recorders or modems.  A
  counter usually counted CPU ticks between two external events (such as when
  the state of an input pin flips between ON and OFF), and a timer usually
  produces an event (mostly a CPU interrupt) when a programmable counter value
  reaches zero.

### Next up: instruction-stepped vs cycle-stepped CPU emulation

I left out the whole CPU part, this will go into its own blog post. The reason
is: all the support chips in a home computer system are usually doing the same
simple work for each tick, they just bump some counters, and in some ticks,
those counters bump other counters. 

A CPU is a much more complex state machine however, while a CPU is
also stepped forward tick by tick in a real computer, the work that happens from
tick to tick is very different.

It is easier (and more efficient) to implement a CPU emulator that is stepped
forward on the opcode level.  Every time the CPU is stepped, a new opcode
will be fetched and executed atomically. There is only a 'before state' and an
'after state', no 'inbetween state', but each of those steps may take a
different amout of clock ticks (for instance on the Z80 an instruction can take
between 4 and 23 ticks, while the 6502 takes between 2 and 7 ticks to execute
one instruction).

This 'instruction-stepped' approach is is currently used by all the Z80 systems
in my emulator, and for most situations this works quite well, but it breaks
down for cases where two high-frequency systems (like the CPU and video signal
generation) need to be synchronized down to single clock ticks. This is for
instance the case in demo-scene graphics demos for the Amstrad CPC:

If a bit in the video chip isn't flipped by the CPU in exactly the right
microsecond, the output image may end up being complete garbage. The problem is
that a single instruction on the CPC can be up to 6 microseconds long, and
those 6 microseconds translate to 24, 48 or even 96 horizontal pixels for the
CPC's 160, 320 and 640 pixel wide video modes (I hope I got that right - for
each 1 MHz video system tick 2 bytes of video memory will be read, and each
byte produces 2, 4 or 8 pixels).  So that's a lot of pixels displayed during a CPU
instruction, and if the timing isn't exactly right, the result isn't just a
small pixel-sized artefact somewhere on the edge of the screen, but a huge
flickering mess.

Anyway... more on the whole CPU topic next time!

