---
layout: post
title: "A Web-API Lament"
---
(as told by a cross-platform game developer)

It may be a sort of Baader-Meinhof effect, but I'm noticing more and more
"native-development" peeps in my twitter timeline who get sucked into "web
development" via WASM, and then are a bit shocked and surprised how weird
everything is compare to creating native realtime 3D applications. And since I've
dabbled a bit in that area as well [for quite a
while](https://floooh.github.io/2012/10/23/mea-culpa.html) I thought: "well gosh,
that's a splendid topic for a blog post!".

## A Trip Through Time

To understand the web of today as a native developer we need to go all the way
back to a time when Flash still ruled supreme for web gaming, but the stellar
growth of mobile phones already cast its shadow over Flash. This must have been
around 2008 (or a few years earlier, 2008 was around the time when the topic
entered my mind).

You see, the end of the Naughties was a time when nothing really made sense in
game development, at least outside of the cushy game console world. PC gaming
would die (this time for realz!), crushed between mobile and console gaming, a
new business model made the rounds which made obscene amounts of money despite
giving games away for free, usually games with laughable production value and
cobbled together by tiny teams in a few weeks (setting quite unrealistic
expections for more "tradtional" game developers trying to find publishing
deals).

Basically, around 2008 or so, if you were an independent PC game developer who
just scraped by without any big AAA titles under your belt, you better got your
game running on the web pronto, and if you were a successful Flash game
developer, better prepare for moving to mobile, because the goose will soon stop
laying golden eggs.

This general migration panic led to a flurry of new developments: suddenly
transpiling C/C++ code to "exotic" platforms like web browsers was all the rage,
several attempts to bring hardware-accelerated 3D rendering to browsers were started,
and a small game engine from Denmark that nobody took seriously (who in their
right mind would write a game engine on a frigging Mac!?) suddenly was the
hot new shit.

From a game developer's perspective (well ok, *my* perspective) it wasn't Steve
Jobs who single-handedly killed Flash, he merely prepared the crime scene by preventing
a flood of cheap Flash game ports to iOS. What *actually* killed Flash with a
vengeance was Unity by supporting iOS early on. Game developers simply didn't
have iOS on the radar in the beginning and were surprised by its success. iOS
was derived from OSX, an operating system which was just as irrelevant for
gaming back then as it is today, and nobody believed that Apple would ever take
gaming seriously (which, in a way, is still true), *especially* not under Jobs.
But "life always finds a way", and Unity (being a "Mac-first" engine) was in the
right place at the right time and the rest is history (which btw was a very similar
situation to the early 2000's when the PS2 exploded in popularity, but the hardware
architecture was so weird that the easiest way to get stuff running on the PS2
was to license the RenderWare engine - which was about as popular back then as Unreal
Engine and Unity are today combined - until EA made the genius move to buy
RenderWare, and the rest is history yet again).

Disclaimer: other people with more insider knowledge might have different opinions
about the why, how and when, but this is essentially how I experienced it back
then as a PC game developer).

Ok - you'll say - that's a lot of trivia about Flash and mobile games from a
time when dinosaurs still roamed the Earth, but that's not of much relevance for
the web today - what's the point?

My point is that this time actually was very critical for the future of web games:

The big transition didn't happen from Flash to HTML5, but from Flash to Mobile,
that's where all the growth was happening (Flash on desktop browsers simply
couldn't grow any more because there wasn't anything left to conquer).

And this exodus of Flash developers to the mobile platforms instead of the
"plugin-free web" very likely had a critical long term effect on today's "web
API landscape": 

- the Flash veterans who had the most experience with game development on the web all left for mobile
- meanwhile the PC game developers who desperately tried to port their engines to
the web didn't have a clue about the specifics of web development
- ...which left all the decision-making to the "traditional" web people, who in turn had no
clue about game development. 

What a f*cking mess...

## Except....

...it's not quite that simple. Around that time of the "Great Migration", two
complete outsider technologies appeared, and against all odds became successful:

WebGL and Emscripten.

It's hard to describe today how extremely "underdog" those two technologies were
when they appeared and what a miracle it is that those very "un-web-like"
technologies were given the time and resources to grow (big kudos to "Mozilla a
decade ago" for this).

WebGL must have been a terrible affront to web people. I can see the Council of
Elders, casting their disapproving gaze over the API specs: Tsktsktsk, this is
not how we do things around here [young
man](https://en.wikipedia.org/wiki/Vladimir_Vuki%C4%87evi%C4%87)! The web is all
about building **content**, and this... "3D API" as you call it - is way too low
level! We need something more declarative, something like HTML and CSS! What's
that? What do you mean "there's no scene graph"? How do would this even integrate
with the DOM if there's no scene graph? What a disgrace!

And besides, [our top men at Google are on it already](https://www.youtube.com/watch?v=uofWfXOzX-g).

Who?

TOP... MEN!

