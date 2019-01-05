---
layout: post
title: Emulators as embedded file viewers
---

Check this out, yo (audio autoplay warning):

<script type="text/javascript">
function start_emu() {
    if (!document.getElementById('emu')) {
        let box = document.getElementById('box')
        box.innerHTML = 'Close the demo<br>'
        box.onclick = stop_emu
        let emu = document.createElement('iframe')
        box.appendChild(emu)
        emu.id = 'emu'
        emu.src = '/files/cpc.html?file=/files/dtc.sna'
        emu.style = 'border:0;'
        emu.width = 512
        emu.height = 360
    }
}
function stop_emu() {
    let box = document.getElementById('box')
    box.innerHTML = 'DTC by Arkos/Overlanders'
    box.onclick = start_emu
}
</script>
<a style="cursor:pointer" id='box' onclick='start_emu();'>DTC by Arkos/Overlanders</a>

This is the Amstrad CPC [Tiny Emu](https://floooh.github.io/tiny8bit/) home
computer emulator embedded via an iframe. If there's no audio you're on a
browser which blocks autoplay, just click or touch into the emulator canvas
to activate the audio. Eventually I'll come up with a better solution for
this.

But isn't it nice how seamlessly and quickly this started? Almost as if
browsers came with a built-in Amstrad CPC emulator :)

Here's what happens:

The Markdown for this blogpost has a bit of embedded JS+HTML which dynamically
adds or removes an iframe (this is just a quick'n'dirty hack, and I'm not
a web programming expert, I'm sure it could be done more elegantly):

```html
<script type="text/javascript">
function start_emu() {
    if (!document.getElementById('emu')) {
        let box = document.getElementById('box')
        box.innerHTML = 'Close the demo<br>'
        box.onclick = stop_emu
        let emu = document.createElement('iframe')
        box.appendChild(emu)
        emu.id = 'emu'
        emu.src = '/files/cpc.html?file=/files/dtc.sna'
        emu.style = 'border:0;'
        emu.width = 512
        emu.height = 360
    }
}
function stop_emu() {
    let box = document.getElementById('box')
    box.innerHTML = 'DTC by Arkos/Overlanders'
    box.onclick = start_emu
}
</script>
<a style="cursor:pointer" id='box' onclick='start_emu();'>DTC by Arkos/Overlanders</a>
```

Creating the iframe causes 4 files to be downloaded:

- cpc.html (2.1 KBytes compressed)
- cpc.wasm (72.9 KBytes compressed)
- cpc.js (27.8 KBytes compressed)
- dtc.sna (129 KBytes, uncompressed)

The 3 ```cpc.*``` files are the emulator, basically what's
coming out of emscripten (the cpc.html file is handwritten, but similar to
emscripten's shell.html).

The ```dtc.sna``` file is an emulator snapshot file, basically a "VM snapshot"
of a running Amstrad CPC instance. Unfortunately the github webserver doesn't
know about the .sna file extension, so this isn't compressed before download.

The emulator is started after the three ```cpc.*``` files are downloaded
(about 100 KBytes), the ```dtc.sna``` file is then loaded asynchronously by
the emulator. So all in all that's about 230 KBytes for several minutes of
entertainment, smaller than many high-res image files on modern webpages (in fact,
the 512x384 JPEG thumbnails that are shown on the Tiny Emulator main page are
often bigger then all the files loaded when clicking on that thumbnail.

How is the emulator so (relatively) small? There's lots of details to the
answer, but the main idea is to handle WebAssembly development like
programming for a very resource-restricted embedded system:

- the programming language is C, not a high-level managed language which
would need a big runtime (C++ or Rust would also work of course, but you
can't use most of the 'high level standard-library goodies' which
differentiate those languages from C anyway, or at least one needs to be
very, very careful to not accidently add bloat)
- a very careful selection of dependencies, a dependency not only needs to 
provide some functionality, but also must do this in a minimal footprint
- 'only include what you use': the emulator I'm using in this blogpost has 
been stripped from the embedded ROM images for 2 other CPC systems, this
reduced the compressed size of the WASM blob from 111 KBytes down to 73 KBytes),
everything is statically linked, and there's no 'dynamic dispatch', so the
compiler can remove dead code, and finally the executable only contains code
to emulate this one system (not a C64, or ZX Spectrum, these are in
separate 'executables')
- some emscripten-specific tricks, like not using the standard jemalloc
allocator, but the slow-yet-small emmalloc replacement (this is ok, since
the emulator only does a small number of allocations at startup)

Here are some rough line-of-code numbers:

- chip emulators (Z80, AY-3-8910, etc...): 3916
- the CPC system emulator (which glues the chips together): 809
- the Sokol headers (for wrapping rendering, audio, ...): 14338
- misc app-level stuff: 1299

The sokol-headers line count contains the code for all platforms and 3D APIs,
the code actually used for the web platform is at most 1/3, so let's say 5000 lines.

So all in all that's about 10k lines of C code, I think that's quite ok
for an app that renders through a 3D API, plays audio and loads data asynchronously
(and on top emulates a complete computer system).

There's a bit more size optimization potential:

- I could replace the WebGL renderer with 2D canvas. This would cut off at least
a few kilobytes from the WASM and JS files, but the downside would be that this
makes portability harder (the emulator can also be compiled as-is for native platforms:
Windows, OSX, Linux and iOS), also 2D canvas would make it impossible to add 
interesting cathode-ray-tube filter effects via shaders.
- The emulator currently contains code for loading tape- and floppy-images,
and emulation code for the floppy controller and disc drive. If only snapshot loading is needed,
removing this code could shave off another KB or two...
- I hope that most of the .js runtime file can disappear or at least be dramatically
reduced in size one day when WASM can call more directly into JS APIs (currently
all calls to WebGL must go through a JS shim for instance)
- I could compile with -Os instead of -O3, but the emulator code is really
sensitive to the optimization level

It's hard to say how much all this would save without trying it out. I'll try
to set aside some time here and there for such experiments.

And finally: **performance**. 

The Amstrad CPC emulation is fairly expensive, mainly
because of the video system emulation which must be more or less cycle-perfect,
otherwise most of the newer graphics demos simply wouldn't work.

On my mid-2014 13" MBP with 2.8 GHz i5 the CPC emulation (in WASM) needs about 3..4ms per 60Hz
host system frame.
This seems to be about the same as more recent iPhones, but on modern Android
phones this can go up to 7..8 milliseconds. Still fast enough to run well within
the 16.6ms frame-budget, but of course power consumption can become an issue.

The same CPC emulation running as native x86-64 code compiled with clang and -O3
takes about 1.5 to 2.2 milliseconds, about twice as fast as the WebAssembly
version. That's a bit on the higher side from my experience (the difference
is usually 1.2 to 1.5x slower for WASM), but it's not unexpected. The inner 
emulator loop seems to be extremely sensitive to small code generation changes
(e.g. more or fewer memory accesses vs being able to keep stuff in registers,
or maybe the WASM code has more cache misses).

Apart from traditional performance optimization, performance would definitely
benefit from making the emulation only as accurate as it needs to be for
specific scenarios. For instance most old CPC games need a much less precise
emulation than modern graphics demos. The CPU wouldn't need to 'tick' with
machine cycle precision, but could run full instructions atomically, and the
framebuffer wouldn't need to be updated after each 1MHz tick, but once per
scanline. This could improve performance about 2x to 3x, but at the cost of a
dramatically reduced emulation precision.

Currently I think that improving performance for mobile devices is more important
than reducing the download size, so I think putting some work into optimizations
(and a better understanding of how my source code can be improved for WASM performance)
makes more sense than shaving off a couple more kilobytes :)
