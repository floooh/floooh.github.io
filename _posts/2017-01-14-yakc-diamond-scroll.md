---
layout: post
title: "YAKC Emu: The Diamond Scroll"
---

The KC85/3 game _**Digger**_ had a 'diamond scroll' effect which looked like this (I failed at 
creating a properly looped GIF, sorry):

![digger-1](../../../images/digger_good.gif)

This scrolling effect was always completely smooth, no matter how many diamonds where on screen.

When I saw this effect for the first time I was totaly flabbergasted and had not the faintest
clue how this works. The 85/3 had no programmable video chip like some Western home computers,
and it could only assign one fore- and one background color to an 8x4 pixel block (slightly
better than the ZX Spectrum which had 8x8-pixel color attribute blocks). There was no way to update
the colors at a 1-pixel vertical resolution, yet the scrolling effect was clearly pixel-wise.
And of course the 1.75 MHz Z80 CPU wasn't nearly fast enough to update the video memory
bytes of the whole screen each frame anyway.

Then my ignorant 80's teenager-mind made a grave mistake: I just glanced over this interesting 
mystery and left it unsolved...

...

...only to have it come back at me 30 years later and laugh in my face when I
decided to write a KC85 emulator :D

When I tested Digger in my emu, it looked like this :/

![digger-2](../../../images/digger_bad.gif)

...time to finally figure out how that effect works! At
least the broken emulator gave me the important clue that the effect was
done with the color blinking feature of the KC85/3 since all the diamonds
are blinking.

The KC85/3 color blinking feature was fairly simple, assigning the color values
from (hex) 10 to 1F to the foreground color repeated the basic 16 colors but
this time with blinking:

![color-blink](../../../images/color_blink.gif)

This was all I cared to know about the blinking as a kid, I didn't see this
as a very exciting feature to use in games.

When writing the emulator, I had to find out how this basic color blinking is implemented,
let's start at the bottom:

I already knew from the programming manual that the blinking was controlled by
the Z80 CTC channel #2 somehow (the CTC is the counter/timer chip of the Z80
family).

By digging into the lower-level KC repair manual and by running my
finger through the wiring diagrams I found out a few more details:

- the CTC has an input-pin for channel 2 called CLK/TRG2, this is fed by the
  (hard-wired) vsync signal from the video signal generator and is triggered 50
  times per second
- the KC operating system initializes the CTC channel 2 as a counter, every
  tick of the 50Hz signal will decrement a counter-value, and when that value
  reaches zero, the CTC output pin 'ZC/TO2' is triggered
- this output pin ZC/TO2 is directly connected back to the video hardware
  and controls the generation of foreground colors by toggling
  a flip-flop (I'll call it the 'color blinking bit' from here on). If the color
  blinking bit is off, foreground colors will be suppressed, and the
  background color is output even for 'foreground pixels'

The operating system initializes the CTC channel 2 blink-counter with a value
of 20 (hex 14), this means that the color blinking bit will flip every 20
vsyncs, or every 400ms ((1000 / 50) * 20), roughly two times per second.

So far so good, I implemented this blinking logic faithfully in the emulator, and this
worked for the normal use cases which manipulated the blinking frequency by writing
a different counter value to CTC channel 2. 

But the counter can at most go as fast as its hardwired 50 Hz input signal, so
the maximum blink frequency can't be higher than 25Hz (since each 50Hz tick
flips the color blinking bit on or off). All in all this appears useless for
that color-scrolling effect in Digger...

**But now the fun part:**

Digger re-programs the CTC channel 2 from counter- to timer-mode. In timer-mode, the
50 Hz input signal is ignored, and instead the internal counter counts down with the
1.75 MHz system clock! 

The counter is initialized to a value of **528**, this means the blinking bit is flipped
after every 528 ticks of the 1.75 MHz system clock, 528 ticks at 1.75 MHz are about
**308 microseconds**, and the [PAL specification](http://martin.hinner.info/vga/pal.html)
says that one video scanline is 64 microseconds long. 

308 divided by 64 is a bit less than 5. Hmmm.....

This means that Digger toggles the color blinking flag (roughly) every 5 video
scanlines:: first the foreground colors are enabled for 5 lines, then supressed
for the next 5 lines, and so on.

This height of 5 lines indeed matches the pixel height of the
foreground/background color stripes in the Diamond scrolling effect! But why
does the effect scroll up, shouldn't it simply divide the screen into stable
horizontal on/off stripes?

The reason the effect scrolls up is because the overall number of scanlines isn't
an exact multiple of the stripe height. The video scanline counter goes up to 312
(per the PAL spec) and then resets to zero, and since the color blinking flag
toggles in 5-scanline-intervals, there's a little remainder which isn't reset at the
start of a new video frame. This timing error causes the blinking to 'run', similar
to a running image on old, broken TVs (hmm, not a very helpful comparison
in the 21st century I must admit).

**What an amazingly clever hack** to turn a 'boring' and hard-wired hardware
feature into an interesting visual effect that was (most likely) never
imagined by the hardware designers!

Ok, so I finally had figured out *how* the effect works and had everything
wired up right in my emulator, but it still only showed a flickering mess.

The reason was that my emulator's video conversion routine - which converts the
video memory of the emulated system into a linear RGBA8 pixel buffer - only ran
once per frame and always converted the entire framebuffer at once.

But within one 16.6ms frame the blink-flag changed its state many times now, while
the video conversion routine only ran once and was completely blind to those
high-frequency changes, it rendered the whole screen with the color blinking
semi-randomly either completely on or off.

And that was the point where I really understood the whole point of those 
'cycle-perfect' emulators, even though the above example by far doesn't
have cycle-perfect requirements, it only needs to be 'scanline-perfect'.

I had to take care of properly synchronizing the different parts of the emulator to
get that high-frequency blinking effect right and I solved this by decoding video scanlines 
one by one, and at just the right time: 

A 64-microsecond PAL scanline takes 112 CPU cycles (at 1.75 MHz), so whenever the CPU
has finished executing 112 cycles, I'm decoding the next video scanline from
the KC video memory to the emulator's backing-framebuffer. Each scanline pass sees
the blinking-flag in the right state, and the result is a properly 'striped'
image if the blinking flag flips with a high frequency - as long as the blink
state doesn't change in the middle of the scanline (I actually never tried what 
happens in this case on a real KC, OMG the possibilities!).

This worked quite well, but two annoying problems remained: I had to add a
'magic constant' to the number of video scanlines to make the scrolling work in
the right direction and the right speed, and when I moved the player character
around, the scrolling speed was different than just standing idle, which
was clearly not the case on real hardware!

The 'magic constant problem' was a precision error in the counter which
triggers the video scanline decoding. Instead of initializing the counter
directly with a number of CPU cycles I did a complicated computation to
convert the PAL vsync frequency into a scanline frequency and that into
CPU cycles. I suspect somewhere on the way some clip-off happened, and
the resulting error added up to 1 or 2 scanlines every frame.

The second problem of the variable scrolling speed was a more subtle bug in the
CTC emulation. The CTC is updated once after each CPU instruction with the
number of cycles that last instruction took, and while updating the CTC
counters I forgot to keep a remainder around for the next round. This is
related to the whole 'instruction-stepped' vs 'cycle-stepped' topic, and why a
cycle-stepped emulation is the better, cleaner approach, especially for systems
with programmable video- and audio-chips like the C64 or Amstrad CPC.

More on that in the next blog post.

So that's it for today. Originally I wanted to write about the emulator's
clock and system-bus implementation, and only use the Digger diamond scrolling
problem as an intro, but as usual that blog post would be entirely too
long :)


