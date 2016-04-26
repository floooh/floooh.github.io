---
layout: post
title: The Great Oryol Spring Cleaning
---

I'm currently right in the middle of a little 'spring cleaning' for Oryol,
deleting obsolete junk and moving things around a bit. Most of this was rolling 
around in the back of my head for several months, but required
some preparation work and also longer-term build system changes.

Here's what's happening:

- I finally removed the **Messaging** module. This was one of those early bad
  design ideas that needed to be killed before they infect the whole project.
  When I started with Oryol I thought that a unified messaging system would be
  essential for any type of game framework. Basically some sort of system to
  asynchronously send messages between different modules, threads, processes
  and machines, with clearly defined protocols, code generation and so on.  I
  used this system for input events, display-change events and asynchronous IO
  requests.  But every time I used this I had a nagging feeling that the messaging
  system was overkill, and a smaller, more specialized, internal solution (as
  in module-internal) would be better, so I gradually started to remove
  dependencies to the Messaging module until I finally ripped the remaining
  pieces out of the IO module last weekend. The next central concept I'm not
  very happy with is the whole ref-counting/smart-pointer/RTTI complex
  (basically: everything concerning lifetime-managed heap objects), I'd rather
  have the whole concept completely out of Oryol, but there are a few places
  where it still makes sense. So maybe next time...
- The **Time** module is only made of 3 small classes (Clock, TimePoint and
  Duration), it works well and hasn't seen much change and it's very likely
  not going to grow much in the future. Since it is so small I have killed
  the module and moved the Time classes into the Core module.
- I have also killed the **Synth** audio module, this was an attempt to create
  a chiptune-like sound synthesizer, it didn't work well for what it was
  actually built for, but I kept it around since it was useful for the KC85
  emulator. But SoLoud was even better, so I moved the KC85 emulator over to
  SoLoud (the old Synth module was based on OpenAL and not very portable).
- I have simplified the **HTTP** module, it has now only HTTPFileSystem
  functionality, the generic HTTPClient class (which was never tested for
  anything but GET requests) has been removed. The old code was quite ugly
  mainly because it was built on top of the Messaging module (for
  instance, IO requests had to be translated to HTTP requests, and HTTP
  responses had to be translated back to IO results, this inbetween layer is
  now gone.  Focusing on HTTPFileSystem functions has simplified the code a
  lot.

That's about it for deleted code, about 4.5k lines killed without
serious loss of functionality :)

The next thing I was planning for a long time was to split Oryol into 
'Core Modules' and 'Extension Modules', where the extension modules
are separate github repos pulled in via [fips](http://floooh.github.io/fips/).
This way a project would pick-and-choose only the modules it wants, and it
is harder to accidently introduce unwanted cross-module dependencies.

This split into separate github modules was actually one of the initial
goals for Oryol, but after I realized that git submodules and subtrees
are a shitty tool for managing external dependencies, fips was born and needed
some time to grow up. It's not perfect as a package manager but it does the 
job well enough now that I am confident in moving non-essential modules into
their own github repos.

The main reason why I do this now and not later is that I delayed the
integration of more 3rd-party libs (like physics, navigation, etc...), and new,
bigger samples building on those libs because I didn't want to 'bloat' Oryol
with non-essential or seemingly redundant modules.

Case in point is that there's now another really nice immediate-mode UI framework 
[nuklear](https://github.com/vurtun/nuklear) (formerly called Zahnrad)
which I really want to integrate, but without moving the UI modules out of
core there would be 3(!) UI modules as part of core Oryol.

So the plan here is this:

- all 'non-essential' modules will go into their separate 'fipsified'
  github repos:
    - all UI modules (imgui + turbobadger)
    - all sound modules
    - all future modules (higher level stuff and more 3rd-party lib integrations)
- the following feature areas will remain in 'Core Oryol' (at least for now):
    - the Core module (app wrapper, memory, containers, strings, 
      time, thread helpers)
    - async IO (with support for local filesystem and web servers)
    - Gfx (3D rendering via platform-specific 3D APIs)
    - Input (keyboard, mouse, gamepad, multitouch)
    - Dbg (minimal debug text rendering)
- there will be a new repo and webpage for 'complex' Oryol samples (all samples
  that require extension modules). All the basic samples will remain in the
  Core Oryol repo/webpage (the 2 sample webpages will be linked somehow so that
  it will be easy to switch back and forth between the core samples and
  extension samples)

The Core Samples will focus more on being tutorials and tests,
while the Extension Samples will be more 'shiny and chrome'.

'Core Oryol' now also has a clearly defined scope (app wrapper, 3D, input, IO)
which makes it easier to decide what should go into Core and what goes into an 
extension module. 

There will be no new major feature areas in 'Core Oryol' added in the future,
on the contrary, more stuff might even be moved outside into extension modules
(there's a lot of apps that don't ever need to load anything, so IO is probably
the next candidate).

And that's it! I'm currently doing the "module exodus" on a branch, so
nothing will break until I make the switch, and I won't switch until I 
have properly updated the README's and sample webpages :)

