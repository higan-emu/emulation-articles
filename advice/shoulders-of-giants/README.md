I believe the advice in this article applies much more broadly than just
emulation, but nonetheless I am going to present this article from said vantage
point.

When looking to begin a new emulator project, where to begin can be very
daunting: is it okay to look at the source code of other emulators? To read
notes that explain how to implement various details? Or is that just copying?
And how do you make a positive difference in the scene?

# My Perspective

When I started on [bsnes](https://github.com/bsnes-emu/bsnes) back in 2004, I did not start from
scratch: I started with the combined wisdom and research of dozens of talented
individuals that helped to reverse engineer and emulate the Super Ninendo
between 1996 and 2004. I had a full eight years of progress at my fingertips:
source code, documentation, forum posts, and bugfix changelogs from the work of
ZSNES, Snes9X, et al. This, to me, was invaluable.

(I also had six years of my own experience in reverse engineering SNES games for
the fan translations I had worked on, which was also a benefit.)

If you search back to the early days of bsnes, you'll find that I was able to
catch up to their general level of accuracy and compatibility within only six
months, in spite of bsnes being the first major article I had written.

To claim that this represents some extraordinary talent of mine would be
facetious: I had a lot of help, and I'm not ashamed of that. I leaned on the
folks who came before me, studied their work, asked them questions, and
benefited greatly from their patience and assistance.

Folks like anomie, TRAC, etc helped make bsnes a reality. I would not be where I
am today without them.

# Success Begets Success

After the first six months of development, it was now my turn to improve the
state of the art. I developed new techniques to more accurately analyze the
cycle timings of the SNES, and I wrote hundreds of test ROMs to suss out
countless edge cases and new behaviors.

After relying on pre-existing knowledge, I had to transition to discovering new
details myself. This required a hardware setup and knowledge of how to write
programs for the SNES. Even though I technically had that from my days of ROM
hacking, such a thing is not necessary to begin writing a new emulator, that can
come later on. These days, for most major retro systems, there is enough
information to create relatively high-quality emulators without ever even owning
the original hardware. Case in point: the MiSTer SNES emulation core was written
by someone using only bsnes' source code as a reference, having never even owned
a Super Nintendo!

# Returning the Favor

bsnes has now been under active development for the past fifteen years. But I
haven't forgotten my origins. Instead, I've strived to give back. You can see my
contributions in pretty much anything in the SNES space today: I have answered
questions for and/or donated source code to Snes9X, SNESGT, Mesen-S, the Super
Nt, etc.

My source code has always been open source, and where required I've even
relicensed it for use in other projects, such as my APU core for Snes9X's
non-commercial license.

And just as I was able to catch up to ZSNES and Snes9X before me in only a tiny
fraction of the time, modern SNES emulators from 2018 onward have quickly been
able to catch up to me.

The Super Nt was created in only nine months, and Mesen-S in only four months.

# Research

As an example, Speedy Gonzales for the SNES was an infamous bug. For around 15
years, no emulator could figure out how to run this game. It would deadlock in
the middle of stage 6-2 for seemingly no reason.

I spent approximately eighty hours of a two-week period reverse engineering the
game, and trying to understand what was happening. It was actually pretty easy
to get the game playable: there were several ways of getting around the problem
area of code. But only one way was the *right* way. Implement any of the other
ways would mean my emulator had two bugs instead of one. Potentially an
incorrect fix would end up breaking a different game in the future.

And so the challenge was ruling out every other possibility through devising a
comprehensive and exhausting set of tests, so that I could be sure I had it
right.

I eventually reached this conclusion: the game was reading from an unmapped
memory address, waiting for a bit to be set that never would be. But by sheer
chance, after thousands of scanlines, an Hblank DMA transfer would fetch a value
with said bit set, and if the cycles aligned just right, this value would remain
on the bus, and the loop condition would terminate.

After this was discovered, I was able to provide the answer to other SNES
emulator developers, who could then implement this behavior within a few
seconds.

This is no different than all of the bugfixes I gleaned by reading through the
old Snes9X changelogs and forum posts about problematic games.

# Cheating?

I know it can seem like it, if you view emulation as taking a test and other
emulators as having the answer key for the test. And by all means, if it is your
ambition to learn *how* to reverse engineer hardware for the sake of it, you
are most welcome to not study what was done before you.

But if your goal is preservation of the original hardware, then it is not in any
way "cheating." It is simply being practical and making use of the resources
that are available to you.

We have limited time in this world, and the hardware we seek to preserve is not
getting any younger, nor any more readily available. Eventually, it will not be
possible to pick up working original hardware to analyze. That is already the
case for some very old and rare systems, and even more modern ones have become
prohibitively expensive and fragile, such as the Atari Jaguar CD.

There's no need to reinvent the wheel: make use of what tools we have, and then
strive to build an even better wheel. Any lingering guilt you have can easily be
absolved once you give back to the community.

# Closing

My hope is that the broader community can see emulation as team effort, and not
as a competition. It isn't about who did it first, or who did it best. It's
about making sure the works of video game studios can live on, and that our
great-grandchildren can, if they so choose, look back and see how video gaming
began.

No one in emulation is a god or a king. No one is more important than anyone
else. We're all in this together, collectively.

Again, you don't have to rely on existing knowledge. You are welcome to forge
your own path. But there should be no shame in standing on the shoulders of
those who came before you.

Thank you for reading, I hope this will help some of you to get started. I look
forward to seeing what you come up with one day soon. Good luck, and don't
forget, we're here for you!
