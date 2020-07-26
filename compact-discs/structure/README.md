In this article, I'll break down the overall structure of data stored on
CD-ROMs, cover what data is currently missing from most CD-ROM image formats,
and propose a new CD image format that obviates the need for CUE sheets to
describe disc tracks, while also providing more complete disc preservation.

I'll start from the highest level structure and then go progressively deeper
into the lesser known details of the format.

# Overview

![Disc Infographic](/images/compact-discs/structure/infographic-0.png)

Compact Discs can store 650MB - 737MB of data on them. The data is written to
discs in a spiral pattern, and the exact maximum amount of storage space is
dependent upon how narrow the spiral is written onto the disc. One cannot make
the spiral too dense, or drives will become unable to read them.

Data is encoded into this spiral via pits and lands, which is roughly analagous
to ones and zeroes, but we'll delve into that more later.

We're going to presume 650MB CDs for the remainder of this article, although the
information is applicable to more dense CDs as well.

# Audio CDs

One 650MB CD holds 74 minutes of audio data in signed 16-bit stereo format at
44.1KHz frequency. This is known as the Redbook audio format.

The disc is divided into 333,000 sectors, each of which contains 2,352 bytes of
data. Every 75 sectors represents exactly one second of audio, thus:

```
333000 sectors / 75 sectors per second = 4440 seconds = 74 minutes
```

The Redbook audio standard specifies a lead-in area, which encodes the disc's
table of contents, or TOC. It also specifies a lead-out area, which tells the
disc player when to stop playing a CD. And it also specifies that there should
be a two-second pregap of silence before each track.

Some audio CDs omit the gaps to allow one song to seamlessly transition into the
next without any silence.

The TOC is used to tell the disc players where each track is located, within
approximately one second of accuracy. CD players read the TOC as their first
step when a disc is started, and they cache this information for track seeking
later on.

CDs can have up to 99 tracks, numbered 1 - 99. Each track can further have up to
99 indexes, numbered 1 - 99 as well. I'm not personally aware of any CDs that
attempt to use track 0, but when it comes to index 0, this is the start of the
track pregap, and index 1 is the start of the music.

The TOC stores only the track numbers, and the individual tracks contain the
index numbers in the Q-subchannel data, which we'll get to shortly.

Some bands got clever with the first track's first index, and would set this
further into the disc. The TOC points each track's index 1, and so a portion of
the track would be skipped. And now by rewinding, you would reveal a hidden
"track 0" of audio. But it's really just audio hiding in the pregap of track 1.

Get used to abuses of the CD-ROM format. They're very common.

# Data CDs

Later on, the Yellowbook standard came along which defined a method of storing
data onto CDs.

But it turns out that CDs aren't all that reliable, and the lower-level CIRC
coding (which we'll get to in a bit) wasn't enough error correction.

And so data CDs split up each 2,352-byte sector into 2,048 bytes of actual data,
a 12-byte sync pattern to identify the start of each sector, a 3-byte address
within the current track, a 1-byte mode specifier, a 4-byte checksum, 8-bytes
of reserved data, and 276-bytes of Reed Solomon Product Code (RSPC) error
correction. The error correction portion is split into 172-bytes of P-parity and
104-bytes of Q-parity. This gives us the following format for each sector:

![Mode 1 sector format](/images/compact-discs/structure/0.png)

```
333000 sectors * 2048 bytes = ~650 MB of storage per disc
```

RSPC is used to provide a higher-level error correction. It can detect damages
in data caused by disc scratches and fingerprint smudges, and can repair some of
the errors. The sector checksum, or EDC is a simple cyclic redundancy check to
ensure that the RSPC-corrected data is valid.

The above is what's known as mode 1. The Yellowbook standard also describes mode
2, which can be used for more data storage when the absolute integrity of the
data is not essential, such as for video data. We gain more storage at the
expense of some error correcting ability:

![Mode 2 sector format](/images/compact-discs/structure/1.png)

```
333000 sectors * 2336 bytes = ~741 MB of storage per disc
```

# ISO images

.iso CD-ROM images are data-only tracks that consist of only the 2048-bytes of
mode 1 data per track. This is the most compact representation of a CD, but also
the one that omits the most data.

It is really only suitable for distributing images to be burned onto CDs, eg
Linux OS releases.

# BIN images

.bin CD-ROM images store 2,352 bytes per sector, and can thus encode both audio
and data tracks (in modes 1 and 2.)

The .bin format still omits subchannel-data, which we will get to soon, and the
lead-in and lead-out portions of the disc.

In its place, .bin images come with .cue files, or CUE sheets, to describe the
table of contents in text form.

# Subchannel data

Now we'll start going lower-level.

CD-ROMs have more than just 2,352 bytes per sector. Every sector is split into
98 F3-frames:

![F3 frame format](/images/compact-discs/structure/2.png)

These F3 frames are where you find the subchannel data. There are eight of these
channels labeled P, Q, R, S, T, U, V, W. Each subchannel gets 12-bytes of data
within each F3 frame. Thus, you must decode an entire sector to get the eight
subchannel blocks of data: the subchannel blocks are split across multiple F3
frames.

The P-subchannel is a very simple bit pattern that is used to identify the start
of tracks. The Q-subchannel data is much more interesting: in the lead-in area,
the table of contents are stored here.

Because the subchannel data is not protected by the RSPC codes (it's at a lower
level on the disc), that means it's not always possible to read back these codes
without errors. The Q-subchannel encodes a 2-byte CRC for each block, and then
the lead-in repeats the TOC over and over again, usually for around 7,500
sectors, so that the disc player can keep reading it until it is able to decode
all of the track starting locations.

The Q-subchannel is also used within the tracks, and this tells the disc player
where the laser is currently reading, both in absolute and relative time, which
is how disc players can display timestamps while playing music.

The Q-subchannel data looks like this:

![Q-subchannel format](/images/compact-discs/structure/3.png)

Broken down further, the data section gives us the following information:

![Q-subchannel data format](/images/compact-discs/structure/4.png)

You can see that each Q-subchannel block encodes the track number, the track
index, the relative time within the current track (in minute:second:frame, or
MSF, format), and the absolute time (for the entire disc.)

The R-W subchannels are user-defined. Generally speaking they are not used, but
sometimes they are used for copy protection purposes (some drives are unable to
write them, making CD copying harder), and sometimes they're used to store
additional data. The CD+Graphics (or CD+G) format stores karaoke song lyrics and
low-quality images in these subchannels, for instance.

# Subchannel parity

You'll note that 12 * 8 is 96, but we described 98 F3 frames per sector. The
extra two bytes are synchronization patterns.

What's particularly interesting about these patterns is that they only exist in
eight-to-fourteen modulation (EFM) format, and are not expressible as 8-bit
values. I'll delve into EFM later, but for now, what's important to note is that
in every image format, these synchronization bits are omitted, yielding 96 bytes
of subchannel data per sector.

# CloneCD images

CloneCD images are identical to .bin images, in that they store 2,352 bytes per
sector, but they also usually include .sub files, which store 96 bytes of
subchannel data per sector.

They also usually include .ccd text file descriptors of the CDs, which is a more
low-level version of a CUE sheet, and is quite proprietary.

# Recap

If we were to combine the sector data with the subchannel data, that gives us:

```
2352 bytes per sector + 96 subchannel bytes per sector = 2448 bytes per sector
333000 sectors * 2448 = ~777 MB of data
```

Our "650 MB" CD is now 777 MB once we've factored in the RSPC and subchannel
data. But we're not even close to finished yet.

# Cross-Interleave Reed Solomon (or CIRC)

Each F3 frame consists of 33 bytes of data: one byte of subchannel data, plus 24
bytes of sector data, plus 4 bytes of P-parity and 4-bytes of Q-parity for the
lower-level of Reed Solomon error correction. This is quite different from RSPC,
and even audio track data gets protected by CIRC error correcting codes.

This is a part where we really don't have any commercially available disc drives
capable of giving us the underlying CIRC codes: the CIRC correction is applied
to the data, and then discarded. Which gives us:

```
98 F3 frames * 24 bytes of data per frame = 2352 bytes of data per sector
```

Let's just presume a reader were to come along that allowed us to rip the CIRC
codes, that would give us:

```
333000 sectors * 98 * 33 = ~1027 MB of data per CD-ROM
```

# F2 frames

F2 frames are F3 frames minus the subchannel data byte.

# F1 frames

F1 frames are F2 frames minus the CIRC error correction codes.

# Raw channel frames

And now we go all the way to the end of this journey:

Pits and lands on a CD aren't really just ones and zeroes: encoding long series
of all ones or all zeroes can cause a CD-ROM drive to fail to read the disc for
all kinds of complicated reasons involving frickin' lasers (no sharks, though.)

To get around this problem, eight-to-fourteen modulation, or EFM, was devised:
the idea is to have a lookup table to encode 8-bit sequences into longer 14-bit
sequences that are meant to prevent having consecutive 1-bits in the output.

Remember the subchannel sync bytes from earlier? They're not in the lookup table
of 0 - 255 output values, which is why we cannot express them as bytes.

Every raw frame includes its own special 24-bit synchronization word to identify
the start of the frames, and every 14-bit encoded EFM value is appended with
another 3-bits of data called merge bits, which are designed to prevent adjacent
EFM codes from both having 1-bits set.

So this gives us:

```
Synchronization:    24 bits
Subchannel data:    14 bits
Sector data:       336 bits (24 bytes * 14 EFM encoding)
CIRC parity data:  112 bits ( 8 bytes * 14 EFN encoding)
Merge bits:        102 bits (34 * 3 bits per merge bit sequence)

24 + 14 + 336 + 112 + 102 = 588 bits per raw channel frame
```

![Frame Infographic](/images/compact-discs/structure/infographic-1.png)

# Final tally

In addition to the 333000 sectors of a disc, the lead-in is usually an
additional ~7500 sectors, and the lead-out an additional ~6750 sectors. The
exact amount varies, and also changes for multi-session discs.

So then we the lowest-level interpretation of a CD is:

```
(7500 + 333000 + 6750) sectors * 98 frames per sector * 588-bits per channel frame = 2.33 GB of data per disc!!
```

Yep, that's right: every compact disc actually holds about 2.33 *gigabytes* of
data. The CD-ROM format is so incredibly unreliable that all of the layers of
error corrections require 2.33 GB to encode 650 MB of usable data.

# Pragmatism

If we can't even rip F2 frames, we certainly can't rip raw frames. And indeed
there's really not much point in doing so. Any disc copy protection scheme
trying to mess with CIRC codes, or worse, EFM codes, would have a really hard
time having *any* drives read the resulting discs.

As amazing as it'd be for preservation, I feel this is overkill even if it were
somehow possible.

What's important is the lead-in data for the table of contents, the subchannel
data for the TOC values and for eg CD+G discs and various copy protection
schemes (eg as used in the Sony Playstation), and the lead-out because why not?
Technically the standard could be violated and data could be placed there, and
it doesn't take much space.

Reading this amount of data is possible with older Plextor drives, which CD-ROM
preservationists have the ability to acquire, although they are quite pricey
these days.

# Proposal

And so finally, my proposal is a new CD-ROM image format: we store the lead-in,
the disc sectors, and the lead-out. Each sector is the 2,352 bytes of data plus
the 96-bytes of subchannel data, forming 2,448 bytes per sector.

```
(7500 + 333000 + 6750) * 2448 = ~810 MB of data per CD-ROM image
```

Because we include the lead-in data, the TOC can be generated by reading its
Q-subchannel. Thus, this format does not require a CUE sheet or CCD file. And
since the subchannel data is interleaved with the sectors themselves, we also
don't need an extra SUB file.

Thus, this format, which I'll just call .bcd for the heck of it (the extension
really isn't important), is a *single-file*. Not bad, right?

# Compression

The disc size is larger due to lots of (usually) predictable data: if the data
is undamaged, then we can generate the RSPC codes even if they're not included
in the image. A compression format could do this work for us, and indeed, if
you've ever heard of the ECM (error code modeler) software, that is exactly what
it does.

We can further also predict standard subchannel data, since P and Q are supposed
to follow known patterns, and R-W are usually unused and zeroed out.

In doing both of these, we could end up with images that are as small as ISO
images, but much more accurate and complete than any format we have today.

# Scrambling

One facet I didn't talk about is scrambling: CDs really don't like long,
repeating sequences, such as all zeroes for silence on a CD. Each 2,352-byte
sector goes through a reversible scrambling operation (just a XOR operation)
which is meant to prevent long runs of repeated bytes, to help prevent the laser
from desynchronizing while reading discs.

I have yet to hear a convincing argument as to why we should rip CDs in
scrambled format, which would seriously harm the compressability of CD-ROM
images, so at this time, my view is that so-called .bcd images should be stored
descrambled, and if an emulator needs scrambled tracks, it can apply the
bidirectional scrambler algorithm to the sector to obtain said data.

For instance, the Sega CD has a control bit that allows the enabling or
disabling of sector scrambling.

# Source Code

It could be interesting to walk through how disc scrambling works, how EFM
encoding and decoding is done, how RSPC and CIRC Reed Solomon error correction
codes are generated, and how they can be used to repair bit-errors in data.

But I feel this article is long enough on its own as a cursory summary of the
CD-ROM disc structure.

I have however implemented most of this into my C++ template library, nall. The
one exception is that I don't currently have a CIRC encoder/decoder, on account
of not having any CD-ROM images ripped at this level to test it against. Well,
that and it's *really* stupidly complicated, even moreso than RSPC, which was
already a nightmare.

In any case, if you'd like to see how the scrambler works, how RSPC works, how
EFM works, how the checksums work, etc, please feel free to take a look at my
source code in nall.

[You can browse my nall/CD code here.](https://github.com/higan-emu/higan/tree/master/nall/cd)

# Closing

Right now, .bcd is just a preliminary proposal of mine and does not represent
any kind of format standard. I've made this article to both explain how CDs are
structured, as well as to start a conversation about how we might improve CD
preservation.

I look forward to hearing everyone's thoughts. Thank you for reading!

# Credits

This time around I'd like to extend my thanks to MerryMage for the help in
understanding the linear algebra required to implement the RSPC coder. Thank you
so much!
