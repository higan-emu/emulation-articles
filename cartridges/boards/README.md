When one thinks about digital backups of old game cartridges, usually what comes
to mind are ROM images: single binary files that are meant to represent all of
the non-volatile (read-only) memory of the cartridges.

But in fact, cartridge-based games are more than just ROM chips: they are entire
circuit boards. If you've played Starfox on the SNES or Virtua Racing on the
Genesis, you've seen the benefit to this approach in person: any circuits that
can fit inside of a given cartridge shell can be used, and so games would often
extend the capabilities of the original hardware, adding effects such as 3D
polygon rendering, enhanced audio, more ROM storage space than originally
allowed, and much, much more.

So how is it that most retro game system emulators can recreate entire games
from just these ROM image files alone? It turns out to be a really complex issue
involving high-level emulation of coprocessors, and headers and checksums to
heuristically determine how to play each individual games.

Sometimes, these heuristics go wrong. And they are universally a source of pain
for the aspiring young emulator developer.

In this article, I'd like to take a deep-dive into what makes game cartridges
tick, and the challenges and limitations of the 'ROM image' approach to digital
archival.

# Inside Cartridges

The Super Game Boy 2 is an SNES cartridge that allows you to play Game Boy games
on your SNES console:

![Super Game Boy 2 cartridge](/images/cartridges/boards/1.jpg)

But how is such a thing possible? Let's examine what's inside of the cartridge:

![Super Game Boy 2 PCB](/images/cartridges/boards/2.jpg)

What you see here a complete Game Boy system built into a single chip, which is
marked `CPU SGB2`. After that, you have an interface between the Game Boy and
the SNES, which also provides many enhancements by allow Game Boy games to
access SNES hardware functionality, the `ICD2-R` chip. Next you have some
dedicated RAM for buffering the Game Boy screen output into a format the SNES
can display, the `Sharp LH5164AN-10L` chip, a region-lockout chip, the
`F411B`, and finally, the "ROM image", which is typically the only contents of
the "Super Game Boy 2.sfc" image, the `SYS-SGB2-10` mask ROM chip.

You also find a Game Boy cartridge slot poking up from the other side, which is
how you connect Game Boy games to the SNES. And you also see a link port
connector on the left, which allows linking the Super Game Boy 2 to another Game
Boy, or even another Super Game Boy 2.

Finally, notice the extra four pins on the bottom left and right sides of the
image: on the SNES, these contain, among other things, stereo audio mixing pins.
The SNES doesn't emulate the Game Boy audio, the Game Boy SoC directly feeds its
audio out to the SNES, to be mixed with the SNES' own audio output, to finally
be sent to your speakers.

Other details: the other side of the board contains its own dedicated clock
(oscillator) which is a 5x multiple of the original Game Boy clock rate, and
inside of the `CPU SGB2` IC, there is a boot ROM that performs Nintendo
licensing validation of the inserted Game Boy cartridge and display of the
splash screen you see when turning on the Super Game Boy. Even the way the ROM
appears inside the SNES memory map is determined based on the wiring of the
cartridge to the pins along the bottom of the PCB.

At the time of this writing, [[bsnes:://byuu.org/bsnes]] is the only SNES
emulator to fully emulate this cartridge. So how does it do it with just the
`SYS-SGB2-10` ROM image file?

# Heuristics

The simple answer is heuristics.

In the case of the SNES, Nintendo required developers to include a 64-byte
internal header that tells you the name of the game, and some basic information
about the cartridge. When you load SNES games, emulators try to decode this
information to determine how to emulate the games.

The title for the Super Game Boy 2 is `Super GAMEBOY2`, and so when a game is
loaded with that title, the code path to emulate the SGB2 is utilized.

In bsnes' case, the emulator contains a complete Game Boy emulator inside of it.
This can be either the Game Boy core from higan or Same Boy, depending on the
version of the emulator used. bsnes also contains an emulation of the ICD-2
interface chip that connects the Game Boy to the SNES.

What happens next is that bsnes looks up a database entry describing the
`SHVC-SGB2-01` PCB, and finds this:

```
board: SHVC-SGB2-01
  memory type=ROM content=Program
    map address=00-7d,80-ff:8000-ffff mask=0x8000
    map address=40-7d,c0-ff:0000-7fff mask=0x8000
  processor identifier=ICD revision=2
    map address=00-3f,80-bf:6000-67ff,7000-7fff
    memory type=ROM content=Boot architecture=LR35902
    oscillator
slot type=GameBoy
```

# Databases

You'll notice that some information is missing here: the ROM size, the
oscillator clock frequency, etc. The reason for this is that multiple games can
often reuse the same PCBs, and so I've separated the game-specific portions of
information from the PCB-specific portions. bsnes has an internal database of
every game I've analyzed (over 1200 games and counting), and so this information
is completed by looking up the game in my games database that is bundled with
every copy of bsnes:

```
game
  sha256:   e1db895a9da7ce992941c1238f711437a9110c2793083bb04e0b4d49b7915916
  label:    スーパーゲームボーイ2
  name:     Super Game Boy 2
  region:   SHVC-SGB2-JPN
  revision: SYS-SGB2-10
  board:    SHVC-SGB2-01
    memory
      type: ROM
      size: 0x80000
      content: Program
    memory
      type: ROM
      size: 0x100
      content: Boot
      manufacturer: Nintendo
      architecture: LR35902
      identifier: SGB2
    oscillator
      frequency: 20971520
```

Finally, we have all of the pieces required and we can emulate the Super Game
Boy 2!

But you may be wondering how bsnes runs all of the games not in its internal
database: and that goes back to the earlier section on heuristics. When games
are not found in the database, the other header fields (ROM size, mapper type,
etc) are decoded, and a best-effort attempt to build a generic board and game
description is performed.

And herein lies the major problem faced when emulating cartridges as ROM files:
heuristics are just best guesses. Not only is the internal header information
incomplete, sometimes it's just plain wrong, perhaps even intentionally as a
crude form of anti-piracy, and sometimes some game systems don't even //have//
internal headers to work with.

From this point in the article, I'm going to talk about all of the ways
heuristics fail us, and why single-file ROM images, while far and away the most
popular form, are not the best way to preserve and emulate game cartridges.

# Problem 1: bodge wiring

Here is the circuit board for the SNES game, Rockman X (Megaman X):

![Rockman X PCB front](/images/cartridges/boards/3.jpg)

This looks simple enough. It's an 8mbit ROM chip and another 4mbit ROM chip,
a 74LS chip to handle the memory mapping, and a lock-out chip, on the standard
`SHVC-2A0N-01` PCB:

```
board: SHVC-2A0N-(01,10,11,20)
  memory type=ROM content=Program
    map address=00-7d,80-ff:8000-ffff mask=0x8000
map address=40-7d,c0-ff:0000-7fff mask=0x8000
```

Easy, right? Well, turn the PCB of the original Rockman X v1.0 release around
and you'll find this:

![Rockman X PCB back](/images/cartridges/boards/4.jpg)

Bodge wiring between one of the ROM chips and the 74LS memory mapper! Why in the
world does every Rockman X v1.0 cartridge produce have hand-soldered wires on
the back of it?

Let's take a look at what the memory map ends up looking like on this cartridge:

```
board: SHVC-2A0N-01#A
  memory type=ROM content=Program
    map address=00-2f,80-af:8000-ffff mask=0x8000
map address=40-6f,c0-ef:0000-ffff mask=0x8000
```

Unlike the standard `SHVC-2A0N-01` configuration, the address map space for
30-3f,b0-bf:8000-ffff and 70-7f,f0-ff:0000-ffff are now left unmapped.

The SNES address map is 24-bits, giving you 128mbit of addressable space. So
what happens when you plug in your Rockman X game cartridge is the 12mbit of ROM
gets repeated. 12mbit isn't a power of two, and so that 4mbit ROM chip gets
repeated twice to give you 16mbit. Then that 16mbit is repeated four times
throughout the memory map. The other remaining addresses are for other portions
of the SNES hardware (RAM, I/O registers, etc.)

What this bodge wiring does is prevent that 4mbit ROM chip from being repeated,
leaving holes in the memory map that do not return any value in read, a state
known as "open bus".

So the question now is, why do this at all? And the answer turns out to be
copy-protection gone awry.

Back in the day, SNES games were pirated using copiers. Supplied with ROM image
files placed on floppy disks, SNES copiers would attempt to simulate a cartridge
being inserted by scanning the internal SNES headers and trying to replicate the
cartridges.

Capcom's developers wanted to thwart would-be pirates, and they knew their game
was only 12mbit. The 70-7f,f0-ff:0000-ffff region was often used by games for
save RAM, and copiers would often provide save RAM for all games under the
reasoning that if games didn't use it, it would just be ignored.

Capcom added checks throughout various portions of Rockman X where the game
would try to write to the 70-7f bank regions, and if it successfully read back
the value written, it would consider the game as being played on a copier, and
do things to subtly frustrate the player. You can read all about this on the
excellent wiki,
[[The Cutting Room Floor:://tcrf.net/Mega_Man_X#Copy_Protection]].

What happened when Capcom produced a large run of games on the `SHVC-2A0N-01`
PCB was that 70-7f now had their smaller 4mbit ROM mirrored there. So in any
instances where they tried writing a byte to the non-existent RAM read back the
same by pure chance that the ROM contained the same byte there originally, their
severe anti-piracy checks would kick in.

How did they make this mistake? For that, look to how SNES games were developed:

![SNES prototype PCB](/images/cartridges/boards/5.jpg)

It was too expensive to produce mask ROMs for development, and so reprogrammable
EEPROM chips were used on special prototype socketed ICs.

The board pictured above takes four 4mbit ROMs, and so when Rockman X was being
developed, the team used three of the four slots. Hence, this board naturally
had it so that the 70-7f banks went unmapped.

Capcom unfortunately didn't realize their mistake until post-production, and a
very costly lesson was undoubtedly learned. This mistake was corrected
immediately in the Rockman X v1.1 batch of games, as well as the US Megaman X
game cartridges.

So what does this have to do with emulation? Simple: there is //absolutely no
way// to know about the presence of this bodge wiring from looking at Nintendo's
internal header.

For many years, SNES emulators would run Rockman X with the 4mbit ROM mirrored,
potentially triggering the copy protection. This likely went unnoticed since
most gamers would be playing the v1.1 Japanese revision, or the English release.

bsnes emulates this correctly by using its internal database:

```
game
  sha256:   2626625f29e451746c8762f9e313d1140457fe68b27d36ce0cbee9b5c5be9743
  label:    ロックマンエックス
  name:     Rockman X
  region:   SHVC-RX
  revision: SHVC-RX-0
  board:    SHVC-2A0N-01#A
    memory
      type: ROM
      size: 0x180000
      content: Program
  note: Custom wiring on PCB
```

I have chosen to denote non-typical PCBs with #n suffixes onto the PCB IDs. In
this case, via `SHVC-2A0N-01#A`.

# Problem 2: incorrect headers

Since real hardware doesn't care about eg Nintendo's internal headers, the
information provided in them is often blank or incorrect when working with eg
SNES prototype cartridges.

One is forced to either hand-edit the ROM images to correct this information,
which changes the data that was originally there, or in the case of bsnes,
include an external manifest file to describe the correct mapping in order to
play many prototype games that have been preserved.

# Problem 3: customizable PCBs

Here is the PCB for E.V.O. - The Search for Eden for the SNES:

![SNES customizable PCB](/images/cartridges/boards/6.jpg)

Ostensibly, this uses the `SHVC-2A3M-01` PCB:

```
board: SHVC-2A3M-(01,11,20)
  memory type=ROM content=Program
    map address=00-7d,80-ff:8000-ffff mask=0x8000
  memory type=RAM content=Save
map address=70-7d,f0-ff:0000-7fff mask=0x8000
```

But note the silkscreened text at U4: MAD-1/R. The MAD-n are a series of memory
mappers that have small differences between them. Nearly all `SHVC-2A3M-01`
boards use the MAD-1. The extremely rare MAD-R has the following difference:

```
board: SHVC-2A3M-01#A
  memory type=ROM content=Program
    map address=00-3f,80-bf:8000-ffff mask=0x8000
  memory type=RAM content=Save
map address=70-7d,f0-ff:0000-7fff mask=0x8000
```

In English: it only supports 16mbit ROMs instead of 32mbit ROMs, and doesn't
mirror the 16mbits into the upper 16mbit regions of the address map.

Does this matter to the emulation of this game? Well, most likely not. But why
take the chance of there being another Rockman X-style copy protection lurking
inside the game?

# Problem 4: save memory types

The Game Boy Advance allows makers to use three types of memory for save game
data: battery-backed SRAM, EEPROM, or Flash memory. EEPROM comes in two sizes
with a protocol that is not directly compatible, and Flash memory comes in three
types with protocols that are also not compatible.

![Game Boy Advance PCB](/images/cartridges/boards/7.jpg)

Nintendo attempted to make life easier for developers by providing code
libraries for each type of memory, to give a standard interface to everything:
just include the relevant library, and call the load/save functions.

But the GBA was unique in being a system that was emulated from a devkit prior
to even being commercially released, and so developers were keen to protect
their investments with strong anti-piracy protection.

Game Boy Advance games do not contain internal headers to identify what type of
save memory is used. What emulators do instead is when you load a game, they
scan the entire contents of the ROM, looking for the identifying strings from
Nintendo's libraries, to try and determine what kind save memory was used: if
the developer used EEPROM, then one would expect to find the `EEPROM_V` string
somewhere inside the ROM image. For Flash, `FLASH(512,1M)_V`. For SRAM,
`SRAM(_F)_V`.

You can probably guess where this is going: game developers edited the strings
to feign being a different type of memory. Some developers included every save
type library in their game ROMs as well. Then, when the emulator or flash
cartridge tried to use the detected save type, it would fail, and trigger the
copy protection, rendering the game unplayable.

Emulator developers have taken to two solutions for this: the first is an
internal database of game checksums used to look up which type of save memory to
use, and the second is a user-selectable save memory type override. The latter
of which is extremely user-unfriendly, as it's quite the ask to expect the user
to know the vendor and product IDs of a flash chip used inside of a cartridge,
when one can't even open GBA cartridges without a special security-bit
screwdriver.

The Game Boy Advance is not alone in this problem: Sega Genesis cartridges
sometimes contained EEPROMs for save memory, and the Genesis internal header
does not specify this at all.

Many other systems in fact have similar problems.

# Problem 5: no internal headers whatsoever

Game cartridges for the Famicom (Nintendo Entertainment System) and MSX computer
systems do not contain any kind of internal headers. And yet due to the
extremely limited capabilities of these machines, a vast array of coprocessors
and custom memory mappers were devised to add more functionality to games.

In the case of the NES, the emulator iNES back in the '90s took to including a
16-byte header onto the top of ROM images to describe a "mapper #" to use when
emulating said game.

Unfortunately, the led to many problems in practice: since there was not a
single person in charge of the standard and tagging of ROM images, often times
many games with different hardware configurations ended up sharing the same
mapper #, despite the two being mutually incompatible. And because this was the
early days of NES emulation, not enough was known about important differences
between seemingly similar games, and so when emulation was improved, the format
was missing vital information.

For one instance, it was assumed that the program ROM for NES games would always
be a multiple of 16 kilobytes ... until Galaxian was encountered, which has an
8 kilobyte program ROM size.

It is also not ideal to tag ROM image files with user-generated metadata, as
this breaks the file hash of the file as a whole (though you can still compute a
consistent hash by ignoring the first 16 bytes, usual file hashing tools will
not have built-in support for such functionality.)

Many attempts have been tried unsuccessfully to correct for the deficiencies of
the iNES header format: from emulators shipping with internal databases, to an
extended NES 2.0 header format designed to be the successor to iNES, to an
entirely new chunk-based header format called UNIF. None have seen widespread,
unified adoption.

And although some emulators get extremely close, there does not yet exist any
NES emulator that play all official and unofficial game cartridges released. The
Chinese aftermarket for Famicom games have added literally hundreds of unique
mappers, for instance, and often there's no way to describe it.

And we keep finding new cases to this day! The recently discovered
[[Konami VRC5:://arstechnica.com/gaming/2019/08/collector-unearths-long-lost-8-bit-konami-games-dumps-them-for-emulation/]]
is a good example of this.

# Problem 6: coprocessor firmware

Here is the PCB to the SNES game, Pilotwings:

![SNES PCB with coprocessor](/images/cartridges/boards/8.jpg)

On this board, one finds the `DSP1` coprocessor. This is a NEC uPD7725 DSP
that is used to performed more advanced mathematical calculations than the SNES
could reasonably handle in real-time. Inside of this chip is custom firmware
used to implement various algorithms.

When ROM images are created for Pilotwings, this coprocessor data is missing.

In fact, it was a substantial effort involving thousands of dollars to decap
these NEC DSP chips and extract the firmware from within them. A good overview
of that is available
[[here:://www.tested.com/tech/gaming/44376-16_bit-time-capsule-how-emulator-bsnes-makes-a-case-for-software-preservation/]],
if you are up for some further reading.

Or you can settle for this electron microscope scan of the insides of the DSP1B
chip:

![Decapped NEC uPD7720 die](/images/cartridges/boards/9.jpg)

So what does an emulator do without this firmware? The only option is a
high-level emulation of the algorithms it provides. This is often quite
inaccurate, and lacks important timing informations: the algorithms tend to
complete instantly, resulting in games running too quickly.

To obtain firmware that is missing from a ROM image would mean having the
emulator ask the user for the location of said firmware, but this has its issues
too: in the case of Pilotwings and Super Mario Kart, different revisions of the
game contain different revisions of this firmware to correct bugs in it.

In Pilotwings' case, a fix to one algorithm resulted in the attract sequence
showing a plane landing to fail, and the plane to infamously crash. How does a
user know which firmware to associate with each game in a case like that? Both
answers are technically valid, and by only providing a single Pilotwings or
Super Mario Kart image, the reality of there being two revisions of each game
are now lost, due to the SNES program ROMs being identical in both revisions of
each game.

# Problem 7: ROM sizes

The Sega Genesis uses a 16-bit Motorola 68000 CPU, and so game ROMs for this
system use 16-bit mask ROMs. Well, almost all games. But take F-22 Interceptor:

![F22 Interceptor cartridge](/images/cartridges/boards/10.jpg)

There exists a version of this game whose second ROM is an 8-bit mask ROM
instead. What happens when you access an 8-bit ROM on a 16-bit CPU? In the case
of the Genesis at least, half of the bits are not set. As a result, every time
this cartridge has been imaged, the resulting ROM file is different due to the
unset being containing different open bus values.

Nothing in the Sega Genesis header format describes the idea of 8-bit ROMs being
a thing: but in practice, as long as the game is aware that only half of the
bits are valid, there's no technical reason they can't do this, and indeed, that
is just what this game did.

# Summary

To reiterate and summarize, here is a list of approaches emulators have taken to
try and solve the disconnect between ROM images and PCB descriptions, along with
their pros and cons:

## Heuristics (first-party official headers)

Great when the games in question always have valid internal headers for all
(even just) licensed games, that describes all possible hardware combinations.
Currently, I've emulated 24 systems and well ... I'll let you know if I ever
find a system where this is actually the case.

## Heuristics (third-party unofficial headers)

Great for games working out of the box with emulators. But with no way to govern
formats like iNES, or to get existing ROM images used by others updated en
masse, there does not currently exist an example of a fully comprehensive
third-party heuristics format.

## Internal databases

The easiest solution for end-users: simply load a game, and it starts playing.

But a dreadful solution for handling unlicensed games, fan translation, ROM
hacks, and homebrew games. These entries are unlikely to be in any internal
databases, and by definition, nothing that came out after an emulator's release
can possibly be in its internal database, so an update mechanism is basically a
requirement.

## External board descriptions

Similar to third-party headers like iNES, the idea is to include a file with the
ROM images that describes the games. You've seen examples of that from bsnes
above, eg from the problem 1 section.

I personally favor this approach because an external file can be stored in
plain-text, and can use an extensible markup format that allows us to add new
corner cases as they are discovered, rather than have a format with limitations
such as iNES permanently set in stone.

However, this approach is also not without its problems: the biggest one is
simply designing the specification. How would one create a markup format that
could adequately describe any arbitrary circuit board configuration, population
and wiring for any game system? And how would one get the hundreds of existing
emulator developers to adopt formats that will often be far more complex than
the needs of their individual systems? For instance, NES emulator authors are
not concerned about Flash memory save type IDs like GBA emulator authors are,
and Genesis emulator authors are not as concerned about memory map layouts
(where things are always linear) as SNES emulator authors are (where things are
banked.)

Further, there's still the issue of what we do when a board description is
released with an error in it. How do we update every copy of the game in the
world with the corrected version?

And finally, there is intensely strong resistance to the idea of storing more
than a single file per game image, even if inside of a ZIP archive. I do not
fully understand the resistance here, but it is a reality we have to face if we
want a ubiquitous solution.

# Closing

So unfortunately, I can't end this article with a solution. It's an open problem
in emulation.
