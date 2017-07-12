---
layout: post
title: What I want from a computer
---
A loose collection of (very IMHO) ideas what a computer system built for
productivity should be like, everything goes, but nothing too crazy. 

### Operating System

The OS itself is reduced to a minimal hardware abstraction layer:

- input
- audio
- filesystem
- networking
- 3D rendering API
- (and probably one or two essential things I forgot)

The OS APIs are exposed as C headers, because C is the lingua franca that all
other languages can talk to. That doesn't mean that the OS itself
has to be written in C.

There's only a simple window composer which can compose rectangles rendered
through the 3D API.

There is no UI framework built into the OS, applications
select their own 3rd-party UI framework (may the best win). Having
no standard UI framework sounds messy, but it is already a reality for
cross-platform apps, which are usually written with Qt or Electron.

Above this basic hardware abstraction layer there's a fork in the road,
one side leads to the dark side, a 'walled garden consumption OS', the 
other side to an 'open productivity & creativity OS'. It doesn't make
any sense trying to combine the two ideologies, it only ends in pain
and suffering for everyone, as Windows 8 has shown.

This is about the creativity/productivity fork:

### Software Installation

A 'friendly' command line package manager like brew, coupled with a very
simple configuration system which allows me to automate system setup (what
software to install, config files, etc...).

Basically the opposite of Windows's sloppy installation system or the
Mac's restrictive App Shop.

The idea is that I want the entire configuration of my machine in a setup script
under version control. Resetting the machine to a clean slate, or setting up
an entirely new machine should not require any manual work.

### Filesystem

Instead of a rigid directory hierarchy I want to add custom attributes to
files (tags and key/value pairs). Some sort of hierarchical tags could be
used to simulate traditional directory hierarchies. The 'filename' is also
just a tag. File management tools and command line shells have realtime
fuzzy-search by file attributes (much like the incremental fuzzy search in
modern text editors works), while at it, drop the whole 'desktop metaphor'
bullshit once and for all.

There are no silly icons (except maybe for apps), only the
*file content* and tags are important.

I'm not sure about an integrated version control system, on one hand
it might make sense, on the other hand there is no consensus what a 
good VCS looks like. So better to let the user decide.

### User Interaction

I want 'friendly, explorable, text-driven' interaction. I want to
start typing anywhere, anytime, and the system or current app should offer me
options through fuzzy-search (of apps to start, files to work on, 
commands to execute and so on).

I want to easily navigate previous actions (through an ever-present context-
sensitive history), and possible next actions (through the fuzzy matching).

Look at the fish shell and Sublime Text's command palette for examples
how this would work. 

Being mainly text-driven doesn't mean there can't be typical mouse- or
touch-driven apps like 3D-modelling or 2D-drawing tools.

### Scriptable UI Apps

All UI apps worth their salt should also offer a language-agnostic scripting
interface for automation tasks, similar to what the Amiga started
with AREXX scripting.

### Conclusion

> Netscape will soon reduce Windows to a poorly debugged set of device drivers
>
> *(Marc Andreessen, 1995)*

That's exactly what *all* operating systems should be IMHO, except for the 
'poorly debugged' part of course :D
