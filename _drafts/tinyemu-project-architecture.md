---
layout: post
title: "Tiny Emu Walkthrough: Architecture"
---

I think the [Tiny Emulators](https://floooh.github.io/tiny8bit/) have settled
into a fairly stable state now where it makes sense to describe their
internals in a little blog post series (no promises that I will actually do a
complete series, but at least that's the idea).

## First a bit of history

...which is important to understand some design decisions I made along the way.

According to github, my venturing into emulator coding started around
the end of 2015, at least around that time was the first commit of the "YAKC" project:

https://github.com/floooh/yakc/blob/master/LICENSE

I dabbled with MAME and other emulators before that, but I really only understood
how emulators actually work after starting to write my own code.

### In the beginning: A C++ framework monolith

YAKC is a multi-system emulator written in a simple C++, implemented on top
of Oryol. Oryol is an "integrated framework" for simple game-like applications,
which makes it easy to write a cross-platform app with simple 3D rendering,
input and asynchronous IO, and an extension module system for integrating things
like UIs, audio or physics libraries:

http://floooh.github.io/oryol/

Using Oryol has some implications on the overall design of the application:
now you're writing an "Oryol application" (whether you want to or not) which
means you'll have to follow Oryol's idea of a specific C++ style, and how a
project is structured (of course you're free to leave the beaten path, but
this will add unnecessary friction). On one hand it's quite easy to get
started with a new application, on the other hand this simplicity is bought
with restrictions and enforcing some rules what the code and the whole
project structure looks like.

With this "framework mindset" firmly planted in my head I started to write
all code for my emulator in the "Oryol philosophy"...

And it worked well! I made good progress, and after I reached my initial goal
(emulating the East German KC85 computers) it was easy to add more system
emulators (like the ZX Spectrum, Amstrad CPC, Acorn Atom and C64).

But after a while, this "integrated framework" approach didn't feel quite
right anymore, somehow it felt "inside-out" of what it should be. Sure, I had
this working multi-system emulator, everything that's important was well
within expectations (e.g. lines of code, performance, executable size). But I
had a nagging feeling that putting all emulators into one executable was the
wrong approach. If I only care about the Amstrad CPC, all the code needed for
the C64 is also in the application, even if it is never running. That doesn't
make much sense, especially for a web application (even if we're just talking
about a few dozen kilobytes).

What I *actually* wanted (even if I didn't know it in the beginning) was a
minimal "runner" for 8-bit home computer games and graphics demos.

The emulator itself should completely disappear into the background, running
the emulator should not feel like opening a heavy desktop or mobile
application (with splash screen and all that crap), but instead it should be
as unremarkable as clicking an image-link on a webpage. For the WebAssembly
version (which I consider the "lead version" which drives most of my design
decisions), this means that every kilobyte counts so that the initial
download and start feels instant.

A related problem became apparent when I tried to embed the YAKC emulator code
into another application (like this Oryol sample: http://floooh.github.io/oryol-samples/asmjs/KC85-3.html).
I didn't have this embedding scenario as a goal from the start, but the
longer I worked on the emulator the more I cared about using at least the
low-level chip- and system-emulator code in other projects. But actually doing
this was quite a bit more hassle than it should be.

The reason was of course that the emulator was created as a "framework
application" first, not as a library. I had made a lot of subconscious design
decisions along the way which were a good fit for writing a monolithic
application, but bad for integrating the code into small specialized
applications.

### The Anarchist Revolution

At around the same time (early 2017) I started thinking more and more about
header-only libraries written in C. The ease of use, and especially the easy
integration of the STB headers (https://github.com/nothings/stb) made a deep
impression on me. Integrating 3rd party code into C and C++ projects is
usually an ugly and tiresome business. With C projects it's mostly complex
build system issues, and with C++ projects the main problems (in addition to
C's build system issues) are typically weird, overengineered APIs and a
"Sorcerer's Apprentice" approach of using every imaginable C++ language
feature.

And then suddenly (for me at least) was this glorious "anarchist
revolution" of the STB headers: just throw all the code into headers, don't
have any sort of build files, don't have a fancy dependency manager with
version pinning, instead copy the required headers right into your project.
And - WORST OF ALL - use the archaic C language instead of the "industry
standard" C++!

Must be a joke right?

And yet it took me maybe 15 minutes replacing an image loading library 
(which I had spent many hours integrating) with a single STB header which even supported
more image formats (https://github.com/nothings/stb/blob/master/stb_image.h), and
this pattern of easy integration and 'it just works' repeated over and over with
this type of small 'drop-in' libries. So my initial scepticism was silenced
by the obvious, pragmatic fact that this approach works much better than the
current "status quo". Suddenly integrating 3rd-party-code wasn't an
frustrating thing to do along the lines of "sure it sucks, but it's less painful then
writing the damn code myself", but something that "just works".

Clearly I wanted those advantages for my own code as well, especially since
those advantages were gained not by adding new tools and complex on top of
the pile, but instead by *removing* stuff from the pile. You don't even need
a build system, not to mention dependency managers for header-only libraries.

### From C++ frameworks to single-file C libraries

This was around mid-2017, when I started with the sokol  headers (https://github.com/floooh/sokol), basically a porting- and
simplification-effort to move the low-level C++ platform-wrapper code that
was "locked away" inside the Oryol framework into dependency-free C headers.
The main idea being: if I just want simple 3D-rendering- or
audio-playing-solution, I shouldn't need to buy into an entire integrated
framework like Oryol. Instead I just want drop one of the sokol headers into my
project, and start coding. This way my code is even useful for extremely simple
hello-world-style tutorial code that can be built directly from the command line
by directly running the compiler, instead of wasting 3 pages showing how to
setup an IDE project first.

The switch to header-libs changed my approach to programming a lot: instead
of carefully planning and writing few small 'proper applications' I started
to 'scribbled' lots of little tools and experiments, mostly because such
small programs require so little upfront work. Just copy a few headers into
your scratch-directory, create a .c file with an empty main() function,
compile with ```cc tool.c -o tool``` and that's it. Maybe 1 in 10 of
those scribbles turn into something more serious where I'm creating
a proper github project and build system files.

Long story short, this 'back to the roots' approach worked so well for me
that I decided to do use the same approach for YAKC: start to split out
the emulator code into C headers, so that it is easier to embed emulators
into other projects than YAKC itself.




<style type="text/css">
.frame {
	display: inline-block;
	background-color: #b6bae6;
	padding: 5px;
	border-radius: 5px;
}
.grid { display: inline-grid; }
.grid.w1 { grid-template-columns: auto; }
.grid.w3 { grid-template-columns: auto auto auto; }
.item {
	border: 1px solid black;
	padding: 5px;
	margin: 2px;
	text-align: center;
	color: black;
	font: 14px arial;
}
.item:hover { color: white; }
.item.w3 { grid-column: span 3; }
.item.system {  background-color: CornflowerBlue; }
.item.main { background-color: Orange; }
.item.chip { background-color: RoyalBlue; }
.item.sokol { background-color: HotPink; }
.item.common { background-color: YellowGreen; }
</style>
<div class="frame">
	<div class="grid w3">
		<a class="item main w3" href="https://github.com/floooh/chips-test/blob/master/examples/sokol/cpc.c">cpc.c</a>
		<a class="item system w3" href="https://github.com/floooh/chips/blob/master/systems/cpc.h">cpc.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/z80.h">z80.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/am40010.h">am40010.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/clk.h">clk.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/i8255.h">i8255.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/upd765.h">upd765.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/fdd.h">fdd.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/mc6845.h">mc6845.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/kbd.h">kbd.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/fdd.h">fdd_cpc.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/ay38910.h">ay38910.h</a>
		<a class="item chip" href="https://github.com/floooh/chips/blob/master/chips/mem.h">mem.h</a>
	</div>
	<div class="grid w1">
		<a class="item common" href="https://github.com/floooh/chips-test/blob/master/examples/common/common.c">common.c</a>
		<a class="item common" href="https://github.com/floooh/chips-test/blob/master/examples/common/common.h">common.h</a>
		<a class="item common" href="https://github.com/floooh/chips-test/blob/master/examples/common/gfx.h">gfx.h</a>
		<a class="item common" href="https://github.com/floooh/chips-test/blob/master/examples/common/clock.h">clock.h</a>
		<a class="item common" href="https://github.com/floooh/chips-test/blob/master/examples/common/fs.h">fs.h</a>
		<a class="item common" href="https://github.com/floooh/chips-test/blob/master/examples/common/keybuf.h">keybuf.h</a>
	</div>
	<div class="grid w1">
		<a class="item sokol" href="https://github.com/floooh/chips-test/blob/master/examples/common/sokol.c">sokol.c</a>
		<a class="item sokol" href="https://github.com/floooh/sokol/blob/master/sokol_app.h">sokol_app.h</a>
		<a class="item sokol" href="https://github.com/floooh/sokol/blob/master/sokol_gfx.h">sokol_gfx.h</a>
		<a class="item sokol" href="https://github.com/floooh/sokol/blob/master/sokol_audio.h">sokol_audio.h</a>
		<a class="item sokol" href="https://github.com/floooh/sokol/blob/master/sokol_time.h">sokol_time.h</a>
		<a class="item sokol" href="https://github.com/floooh/sokol/blob/master/sokol_args.h">sokol_args.h</a>
	</div>
</div>

Bla bla bla...