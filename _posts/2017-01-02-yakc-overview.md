---
layout: post
title: "YAKC Emu: Architecture Overview"
---

For my 2016 years-end vacation I didn't feel like starting something completely new, 
so I worked a bit on my [YAKC 8-bit emulator](https://github.com/floooh/yakc),
the asm.js version is [here](https://floooh.github.com/virtualkc). 

I mainly wanted to learn more about the two popular Western Z80-machines **ZX
Spectrum** and **Amstrad CPC**, so I spent most of the time extending the 
emulator for those two systems.

The Speccy and CPC don't quite fit into the emulator's current theme of 'obscure East-German
home computers', but there's still a connection to the East:

The ZX Spectrum had a very simple architecture and could be built with standard
microchips that were mass-produced in Eastern Europe. In East-Germany it were
mostly hobbyists who built their own ZX clones, but in the
Soviet Union (and mid-90's Russia), and in Poland and Romania, Spectrum compatible 
machines were factory-built. Unfortunately those 'proper'
Eastern-European Speccy clones weren't available in Eastern Germany so
I could never try one out myself.

The Amstrad CPC has a more interesting connection to the East: right on time
for the fall of the Iron Curtain in 1989, East Germany presented the [KC
Compact](http://www.cpcwiki.eu/index.php/KC_Compact), a CPC-compatible machine
somewhere between the 664 and 6128. The CPC was much more complex than 
the Speccy, it required a custom gate-array chip designed by Amstrad, an
MC6845 video display controller, and an AY-3-8912 sound chip. Only the
video display controller chip was built as a clone in Eastern Europe. The custom 
Amstrad gate-array had to be 'emulated' with a couple of different chips that
were more easily available, and the sound chip was a Taiwanese import 
(which was extremely unusual, it would be interesting to know the story
how those chips made their way to Eastern Germany).

But with the reunification on the horizon the KC Compact was too late, only
very few were built and sold. It was crushed by the 16-bit Amigas and Ataris
flooding into the East.

When I added support for the Spectrum and CPC I also did a pretty
thorough source code cleanup which should make it much easier to add support
for new 8-bit machines in the future (even non-Z80 based systems, and systems
with a lot of custom logic), and I think now is a good time to finally start
a little blog post series which dives into the implementation details of the
emulator.

### YAKC Birds Eye View

The entire emulator package is split into 3 components:

- **Emulator Core**: This is the actual emulator written in a simple C++ without dependencies. 
The main design goals for the emulator core are to be small and fast, easy to extend with 
new emulated systems, and easy to embed into other projects (for instance here's the
emulator embedded into a simple 
[3D scene demo](http://floooh.github.io/oryol-samples/asmjs/KC85-3.html))
- **Platform Wrapper**: The emulator core doesn't know how to get keyboard input,
or how to render the emulator's video and audio output, this must be provided by the Platform Wrapper
component. The YAKC project comes with an [Oryol](https://github.com/floooh/oryol) 
wrapper, but it should also be simple to use something else (like SDL).
- **User Interface**: the User Interface is an optional component sitting on
top of the Platform Wrapper and Emulator Core, it allows to boot the emulator into 
the different emulated systems, load software into the emulator, and provides tools
like a CPU debugger, memory editor, and various custom chip debugging windows. This
is implemented with [Dear Imgui](https://github.com/ocornut/imgui)

I'm trying to keep the whole emulator small and bloat-free, mainly to keep the 
emscripten-compiled asm.js version small, and so far this looks quite good.

With User Interface and Oryol Wrapper, the 'executable' sizes are as follows:

- OSX, 64-bit: 1.2 MB uncompressed, **496 KB** compressed (gzip --best)
- asm.js: 1.7 MB uncompressed, **360 KB** compressed

...and without User Interface (but with the Oryol Wrapper):

- OSX, 64-bit: 818 KB uncompressed, **322 KB** compressed
- asm.js: 1.2 MB uncompressed, **206 KB** compressed

Those sizes include the code for 5 different emulated system families with 13 different models:

- the **KC85** main line (KC85/2, KC85/3, KC85/4)
- the **Z9001** family (Z9001 and KC87)
- the **Z1013** family (Z1013.1, Z1013.16, Z1013.64)
- **ZX Spectrum** family (48k and 128k)
- 3 **CPC** models (Amstrad CPC 464 and 6128, and the KC Compact)

### The Emulator Core

This is where all the interesting stuff happens, and most of the blog
posts in the series will describe the internals of the Emulator Core. For now,
I'll only talk about how the Emulator Core looks from the outside.

There's a single frontend class **'yakc'** which allows to control the entire emulator.

To get a working emulator, the following steps are required:

1. Initialization and ROM image registration
2. Booting up an emulated system
3. per frame: step the emulator forward
4. per frame: render the emulator's video output
5. request and play audio output
6. provide keyboard and joystick input to the emulator

#### 1. Initialization

This only needs to happen once by calling the *yakc::init()* method, taking
C-function pointers to three externally provided functions to print
a failed-assert log message, and to allocate and free memory.

Here's an example which uses Oryol functions. The Log::AssertMsg() function has
a matching signature, so it is assigned directly as a function pointer. And in the 
2 cases where the parameter signature doesn't match, an indirect call through a 
C++11 lambda is used:

```cpp

ext_funcs funcs;
funcs.assertmsg_func = Oryol::Log::AssertMsg;
funcs.malloc_func = [](size_t s) -> void* { 
    return Oryol::Memory::Alloc((int)s); 
};
funcs.free_func = [](void* p) { 
    Oryol::Memory::Free(p); 
};
this->emu.init(funcs);

```

In the next step, ROM images must be registered with the
emulator. The emulator comes with 2 embedded ROM images to boot into
the KC85/3, but for all other systems, the ROM images must first
be loaded by the Platform Wrapper.

To register the embedded KC85/3 ROM images, no special loading code is
required since the ROM data is directly embedded as C arrays (called
dump_caos31 and dump_basic_c0). The *yakc::add_rom()* method takes
a ROM image identifier, a pointer to the raw ROM image data, and the size in bytes:

```cpp

// register the built-in KC85/3 ROM images from C-arrays:
this->emu.add_rom(rom_images::caos31, dump_caos31, sizeof(dump_caos31));
this->emu.add_rom(rom_images::kc85_basic_rom, dump_basic_c0, sizeof(dump_basic_c0));

```

For the other emulated systems, the ROM images must be loaded and registered
before booting the emulator. For instance the KC Compact requires one OS ROM
image, and one BASIC ROM image, which are loaded through Oryols's HTTP
filesystem in the following example code:

```cpp

// load and register the KC Compact OS ROM
IO::Load("rom:kcc_os.bin", [this](IO::LoadResult res) {
    this->emu.add_rom(rom_images::kcc_os, res.Data.Data(), res.Data.Size());
});

// load and register the KC Compact BASIC ROM
IO::Load("rom:kcc_bas.bin", [this](IO::LoadResult res) {
    this->emu.add_rom(rom_images::kcc_basic, res.Data.Data(), res.Data.Size());
});

```

The emulator can be asked whether all the required ROM images for an emulated
system have been registered:

```cpp

if (this->emu.check_roms(device::kccompact)) {
    // it's ok now to boot into the KC Compact
}

```

#### 2. Booting

After the one-time init and ROM registration, the emulator can be booted into a specific
system configuration with the *yakc::poweron()* method:

```cpp

// boot into the KC85/3 with the CAOS 3.1 operating system:
this->emu.poweron(device::kc85_3, os_rom::caos_3_1);

```

The method *yakc::reset()* performs a system reset. What happens in reset depends on the
emulated system (for instance some systems preserve the RAM content, while on other
systems there's no difference between a reset and a complete power cycle):

```cpp

// reset the current emulated system:
this->emu.reset();

```

Finally, the *yakc::poweroff()* method switches off the emulated system. This should
also be called before booting into a different system:

```cpp

// switch off current system, and boot into the KC Compact
this->emu.poweroff();
this->emu.poweron(device::kccompact);

```

Since the KC Compact cannot be booted into different ROM images, the second
parameter to the *poweron()* method can be omitted for this system.

#### 3. Stepping the Emulator

The method *yakc::step()* is used to run the emulator for a given amount of time. It 
is called once per frame, usually 60 times per second.

The *step()* method takes 2 parameters, the first parameter is the
frame duration in microseconds (at 60fps this would be around 16666). The emulator will
convert the frame duration into CPU clock cycles and run the emulation for this
number of cycles before returning. 

The second parameter to the *step()* method is less straightforward. It is a 
clock cycle counter from the audio wrapper and is used to precisely synchronize 
the emulator with the real-time audio playback. I have arrived at this solution after a lot
of trial and error to get audio emulation 'right', and the topic is interesting
enough for its own blog post. So let's skip the details for now:

```cpp

// once per wrapper-application frame:
int frame_time_in_micro_seconds = ...;
uint64_t processed_audio_cycles = ...;
this->emu.step(frame_time_in_micro_seconds, processed_audio_cycles);

```

The emulator will now happily do its thing, booting up into the operating system,
and then wait for user input. But we can't see or hear a thing yet, and
we can't interact with the emulator through keyboard input.

#### 4. Render the Video Output

The emulator continuously updates a flat RGBA8 framebuffer which is usually copied once per
frame into a 3D-API texture and then rendered as a fullscreen-rectangle.

As said before, the emulator itself doesn't know how to perform this copy-and-rendering
step, this must be performed by the Platform Wrapper. The emulator core itself
just provides a method to access the RGBA8 framebuffer content and its dimensions:

```cpp

// get pointer to and dimensions of RGBA8 framebuffer 
int out_width, out_height;
const void* rgba8_ptr = this->emu.framebuffer(out_width, out_height);
if (rgba8_ptr) {
    // copy data into a 3D-API texture and render as fullscreen rectangle
}

```

The *yakc::framebuffer()* method may return a nullptr if the emulator is currently
in poweroff state, and the returned pointer and dimensions may change every time 
the *framebuffer()* method is called (although in reality this happens only
when booting into a different system).

#### 5. Request and Play Audio Output

Getting glitch-free audio out of the emulator turned out to be much more
complicated than expected. I went through a lot of trial-and-error and a few
complete rewrites until I had a solution which even works with WebAudio as backend.

The main problem is that the audio data generated by the emulator is completely
unpredictable but must be played back with a very low latency. Each of those
alone isn't such a big deal, but unpredictability *and* low-latency is
not trivial. This means that the audio backend needs to use a very small sample
buffer size (to keep latency low), but this makes the whole system very
sensible to buffer underruns where not enough generated audio data is ready when
needed by the audio backend, and the smallest seemingly unrelated
events like switching to a different browser tab might throw the emulator
and audio backend out of sync, producing audio glitches.

The solution I came up with is that the audio backend ultimately controls the emulator's speed. 
Even though the emulator is ticked forward by the per-frame function of the host application,
the audio system defines a small time window which the emulator must not leave.
If the emulator is behind or ahead the audio time window, it will execute more or fewer 
cycles per frame to get back into that window. In the end this means that the audio 
system controls the speed of the emulator with a higher priority than the application's 
framerate.

The audio interface between the emulator and the audio playback system is very simple,
just a single callback method *yakc::fill_sound_samples()*. This is called by the
audio backend with a pointer to a memory buffer which must be filled by the emulator 
with new audio samples:

```cpp

    /// fill sample buffer for external audio system
    void fill_sound_samples(float* buffer, int num_samples);

```

This is the only method in the whole emulator package which might be called from
a different thread.

In the Platform Wrapper layer, I'm using [SoLoud](https://github.com/jarikomppa/soloud)
as medium-level multi-platform audio playback solution. This turned out to be 
a very good choice for the emulator since it is easily extensible with new
'audio source' classes, having that much low-level level control was
very useful for the tight synchronization required between audio playback and
emulator speed. SoLoud also comes with a fast LowPass filter which is great 
for getting that typical 'cheap TV speaker' sound.

#### 6. Providing Keyboard and Joystick Input 

80's home computers typically support keyboard and joystick input.

Due to cost-restrictions many home computers didn't support the
standardized PC-style keyboard layouts which makes it quite tricky to map today's 
keyboard input to emulated systems. For instance the Z1013 only had 32 keys, 
with 4(!) shift keys, but no up/down keys (only left/right), the ZX Spectrum had
a similar non-standard keyboard layout.

The solution I choose for the YAKC emulator is to feed highlevel ASCII codes into the 
emulator instead of lower level hardware key codes. This has the advantage that text
input works 'intuitively' for most host platforms and international keyboard layouts,
but often doesn't work well for controlling games, since the physical key layout may 
be very different, and more importantly, pressing multiple keys at the same time 
isn't supported.

Only one digital joystick is emulated currently, joystick directions are hardwired 
to the cursor keys, and the joystick fire button is mapped to the spacebar.

So all in all, the emulators input interface needs a bit of work in the future.

Right now, input is provided through the method *yakc::put_input()*, which takes 
two parameters: one 8-bit ASCII code, and one 8-bit joystick mask (bits 0..3 for the
4 directions, and bit 5 for a single fire button).

The *yakc::put_input()* method should be called once per frame. If no key is pressed,
the ASCII code parameter must be 0:

```cpp

uint8_t ascii = ...;
uint8_t joy0_mask = ...;
this->emu.put_input(ascii, joy0_mask);

```

#### What's Next

I guess that's enough for today :) My plan is to have a look at the Emulator Core next,
starting with the lowest-level layer (memory system, clock, and the system bus),
and then continue with the CPU and support chip emulation and finally how to build
a complete system by connecting chips through the system bus. After that there should
be 2 dedicated blog posts about the video and audio systems.
