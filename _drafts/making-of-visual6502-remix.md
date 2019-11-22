---
layout: post
title: "Making of Visual 6502 Remix"
---

![Visual 6502 Remix Logo](/images/visual6502remix.png)

I made a little 'accidental' project in the last few weeks:

[Visual 6502 Remix](https://floooh.github.io/visual6502remix/)

I call it 'accidental' because I hadn't planned this at all until I the
day I wrote the first line of code (usually I ponder such things for months
or even years in the back of my head, and only few of those things pondered I
ever start working on). 

In hindsight, it all started with this LC-80 emulator (which was the
computer I learned programming on). Better turn down the audio volume before
clicking the link since the boot sequence starts blaring the Popcorn song
which you won't get out of your head for *days*:

[https://floooh.github.io/tiny8bit/lc80.html](https://floooh.github.io/tiny8bit/lc80.html)

Since this LC-80 couldn't be connected to a TV but only had a small 6-digit
LCD display, I decided to use the available screen space with a visualization
of the motherboard so that it's possible to actually see how the computer
works.

This project was quite interesting but revealed a 'weakness' in my CPU emulators.
Even though they enable a 'cycle perfect' emulation, they cannot be 'cycle
stepped'. If you open the debugger window in the above emulator and start
single stepping, you can only inspect the motherboard state after a full
instruction which hides all the interesting stuff that happens *during*
execution of an instruction. The reason for this is how the CPU emulation
is currently implemented. The CPU takes a central role and 'ticks' the
emulated system forward. This is different from real hardware where
the CPU is ticked by a clock signal like any other chip in the system.

So I started with a proper 'cycle-stepped' CPU emulator. First the 6502, because
that has an easier instruction set architecture. This work in progress is
currently in a branch in the [chips repository](https://github.com/floooh/chips/blob/d2aa188913a60f95cc71694669cca84349fbc3f1/chips/m6502x.h#L508).

While toiling along on that cycle-stepped 6502 emulator I received a very helpful
pointer to the [perfect6502](https://github.com/mist64/perfect6502) project on Twitter:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">The 6502 emulator for <a href="https://github.com/tom-seddon/b2">https://github.com/tom-seddon/b2</a> works this way. One nice side-effect is that it&#39;s possible to use <a href="https://github.com/mist64/perfect6502">https://github.com/mist64/perfect6502</a> to run automated cycle-by-cycle comparisons with the gate-level simulation.</p>&mdash; Tom Seddon (@tom_seddon) <a href="https://twitter.com/tom_seddon/status/1188199062266433541?ref_src=twsrc%5Etfw">October 26, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

I used [visual6502.org](http://visual6502.org/JSSim/index.html) 'manually' in the past
to verify my older 6502 emulators, but having a C port of the transistor simulation
is *extremely* useful for automated tests.

...and while integrating the *perfect6502* simulation into my emulator tests I 
couldn't stop thinking about how this C implementation of *visual6502* would
enable to run the 6502 simulation in WebAssembly, and thus was born:

# Visual 6502 Remix
The idea is to take the chip layout data which was reverse engineered by the
[visual6502 project](https://github.com/trebonian/visual6502), the [C rewrite of the transistor-level simulation](https://github.com/mist64/perfect6502), [Dear
ImGui](https://github.com/ocornut/imgui) for the UI, the [Sokol
Headers](https://github.com/floooh/sokol) for browser integration, throw it
all into the software mixer and create a 'remix' of the original visual6502
project running mainly on WebGL+WASM, but which can also run as a 'traditional'
native application outside the browser.

After about 3 weeks of 'precious spare time work' the result looks like this:

![Visual 6502 Remix Screenshot](/images/visual6502remix_screenshot.png)

Link to the WASM version: [https://floooh.github.io/visual6502remix/](https://floooh.github.io/visual6502remix/)

Apart from the 'standard features' known from visual6502 to step the
simulation and record a trace log, there's now also an integrated assembler.

And the entire application runs at display refresh rate at all times (at
least on 'typical' desktops and notebooks, I didn't care about mobile
devices so far).

I worked with a slightly different mindset from my usual work on emulators
and the Sokol headers: instead of focusing on writing somewhat clean 'bedrock
code' I used a somewhat quick'n'dirty "application authoring" approach where
the focus is on "getting the damn thing done". If there was an existing
library or project to solve a problem, I used this instead of creating my own
solution. If I couldn't find a project to solve any non-trivial problem, I
rather left that part out completely instead of "wasting time" writing my own
solution.

After all this was only meant as a small intermission project before I 
could go back to the cycle-stepped CPU emulators.

The first step was playing around with the:

# Chip Visualization

The chip rendering of visual6502.org look quite complex at first glance which
might be difficult to render in 'realtime':

![Visual 6502 Chip Layoyt](/images/visual6502_layout.png)

There are several layers rendered on top of each other, and the individual paths on
the chip must be highlighted while the chip is running.

On the other hand, the 6502 only has around 3500 transistors (3510 to be correct),
which is a very manageable number for realtime rendering even if the actual
vertex numbers are amplified by a factor of 10 or 100 due to the complex path
outlines.

First I had to figure out how visual6502 stores and renders the chip layout data.
Turns out it's fairly simple:

The 'path outlines' are called segments, and they are defined in a JS file called 
[segdefs.js](https://raw.githubusercontent.com/trebonian/visual6502/master/segdefs.js).

Here's a snippet from that file:

```javascript
var segdefs = [
    ...
    [   0,'+',1,5392,7804,5392,7757,5357,7757,5357,7804],
    [   0,'+',1,5390,8144,5390,8101,5356,8101,5356,8144],
    ...
    [  93,'+',5,8403,6126,8403,6062,8422,6062,8422,6126],
    ...
    [ 710,'-',5,1285,874,1285,850,1366,850,1366,874],
]
```

After a bit of pondering over that data and looking at the other
data files those numbers started to make sense:

The first number in each segment (e.g. 0, 93 and 710) is the 'node index',
each 'node' being a collection of connected segments. Each node can either
currently either be switched on or off, and the initial state is the '+' or
'-' following the node index.

The next number is the chip layer where that segment is placed. The layer
number is between 0 and 5.

The remaining numbers which are between zero and 10000 are 2D integer coordinates
of the actual segment's closed polygon outline in [x,y] pairs. So the first line:

```javascript
    [   0,'+',1,5392,7804,5392,7757,5357,7757,5357,7804],
```

Describes a 2D polygon made of 4 vertices:

```
[x=5392,y=7804] => [x=5392,y=7757] => [x=5357,y=7757] => [x=5357,y=7804]
```

The nodes formed by those segments range from very simple like this (highly zoomed in):

![Simple 6502 Node](/images/visual6502_simple_node.png)

...to nodes spanning the entire chip, like this (the 'cclk' node):

![6502 cclk Node](/images/visual6502_cclk.png)

Now that I figured out what to render the first step was getting the data from
Javascript into something that can be handled by C. For simple scripting
tasks like this I usually start with a little python script, basically 
translating the Javascript file into a C header.

Then write a small Sokol + Dear ImGui program to render polygon outlines
through ImGui's ImDrawList class (ImDrawList can only render filled *convex*
polygons, so I settled with non-filled polygons).

At this point I didn't know yet whether anything serious would come out of
those experiment, so far it was only a little coding doodle. But the first
results were promising:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">OMG looks like it&#39;s possible to render the Visual6502 data set with Dear ImGui at a smooth frame rate! And that&#39;s just a quick adhoc experiment without any optimizations.<br><br>...I think the next steps are inevitable :D<br><br>(cc <a href="https://twitter.com/ocornut?ref_src=twsrc%5Etfw">@ocornut</a>) <a href="https://t.co/7hSIEaa2gM">pic.twitter.com/7hSIEaa2gM</a></p>&mdash; Andre Weissflog (@FlohOfWoe) <a href="https://twitter.com/FlohOfWoe/status/1189618562148847621?ref_src=twsrc%5Etfw">October 30, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

Even though the rendering performance via ImDrawList was 'good enough' in a
native application rendering through Metal even on my 5 years old laptop with
Intel GPU, this approach is quite CPU-heavy. Each frame, 300k vertices are
scaled and translated on the CPU, and around 1.1 million indices are written.
This translates to about 11 MBytes of new data each frame to be uploaded
to the GPU, and when I tried the same code running in WASM+WebGL, performance
tanked.

But even ancient low-end GPUs running WebGL don't have a problem rendering
that amount of data as long as it's static and doesn't change each frame, so
the next step was to leave ImDrawList behind and instead preprocess the
segment data into static vertex buffers.

So back to my offline python script which converts the original python script
which converts the visual6502 segment data into C: instead of straight Javascript-
to-C conversion the script would now preprocess the segment data into something
that can be fed straight to the GPU as static vertex buffers.

This was also the right time to properly triangulate the data, so that
filled convex polygons could be rendered instead of just outlines. In the
project's spirit of doing things quick'n'dirty I looked around for an existing
python triangulator and found it here:

[https://github.com/linuxlewis/tripy](https://github.com/linuxlewis/tripy)

...it works, but this is also where python is showing its limits. While the 
previous simple text translation was nearly instant, it now takes nearly
half a minute to chew through all the data. It's ok though because this translation
only happens once.

As expected, this fixed WebGL's rendering performance. The whole chip
visualization is rendered in 6 draw calls (one per layer), all data is static,
and the amount of data the GPU needs to work through has been reduced as well
because I could switch to a packed vertex format with 8 bytes per vertex instead
of ImGui's 20 bytes per vertex.

But I still couldn't eliminate dynamic data completely: Connected segment
groups ('nodes') need to be dynamically highlighted to indicate on/off state. But
the number of individual segments is much smaller than the number of vertices
needed to render the chip layout:

There's only around 1800 of such nodes which can change state, and that's
a very manageable amount of data to update each frame:

Instead of updating all vertices each frame, only a small 2 KByte texture
needs to be updated and uploaded to the GPU now. The node index had be stored
with each vertex during the offline-preprossing step already.

So compared to my initial naive approach of uploading 11 MBytes of data each
frame to the GPU, this new approach has reduced the amount of per-frame
dynamic data nearly 6000x, and reduced the required CPU work each frame for
rendering to nearly zero (we'll need that CPU perf again later for running
the actual chip simulation).

It's interesting to note that most of the 'subjective' performance increase
compared to the original visual6502.org is (most likely) coming from this
entirely different rendering approach and not because of the performance
differences between Javascript versus C compiled to WASM. As far as I have
seen looking through the code, Visual6502 renders polygons through 2D canvas,
which is closer to my intial 'naive' attempt using ImDrawList.
