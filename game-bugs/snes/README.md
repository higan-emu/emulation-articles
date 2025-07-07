These are bugs that exist in SNES games and which occur even on original
hardware.

# Dirt Racer

![Dirt Racer bug](/images/game-bugs/snes/dirt-racer.png)

This game will freeze on the startup splash screen sometimes when SNES WRAM
contains zero bits in certain locations.

SNES WRAM is non-deterministic at startup, and so there is a chance this bug
will occur on real hardware.

# Heisei Inu Monogatari: Bow Pop'n Smash!!

![Heisei Inu Monogatari bug](/images/game-bugs/snes/heisei-inu-monogatari.png)

When choosing Taisen mode, there is a flickering black line at the top of the
screen. This line is usually cut off by CRT television overscan, but can be seen
with video capture card devices.

What happens is the game takes too long during certain Vblank sequences and
misses the start of the next frame, enabling the display one scanline too late,
but only occasionally, hence the flickering.

# Hurricanes, The

![The Hurricanes bug](/images/game-bugs/snes/hurricanes.png)

During the attract sequence for level 12, sometimes a row of bad tiles are seen
in the above scene in the background.

This is caused by the game not initializing WRAM before transferring data into
VRAM for BG2. If an emulator randomizes RAM at startup as real hardware would,
this bug will be seen under emulation.

The bug has a random chance of occurring on original hardware.

# Magical Drop

![Magical Drop bug](/images/game-bugs/snes/magical-drop.png)

In Tokoton mode, losing the game will sometimes soft-lock, preventing you from
entering your initials and then trying again.

This bug is caused by the game not initializing the SNES DSP registers at
startup. The register values are non-deterministic at SNES startup, and when the
wrong value ends up in certain registers (PITCH and ENVX), this bug will
surface.

This bug is somewhat uncommon to trigger on a real SNES, but multiple
confirmations have shown that it can and does happen on real hardware.

# Super Bonk

![Super Bonk bug](/images/game-bugs/snes/super-bonk.png)

During the attract sequence, there is a chance that the scripted input sequence
will become desynchronized, and Bonk will get stuck in front of the
jet-roller pipeline sign.

This happens due to the natural variance in the SNES CPU and APU oscillators.

Depending on the SNES, the attract sequence will either always work correctly,
always break, or only periodically break. We've seen all three in practice.

As emulators are deterministic in nature (for the sake of tool-assisted
speedruns and bug reproducability), this bug is likely to either always occur or
never occur under emulation.

# Zenkoku Koukou Soccer 2

When a match is paused,
the game reads the joypad registers in a tight loop,
even during the auto-joypad polling process,
and so can get confused about which buttons are actually pressed.
As a result,
when you hold the Start button to pause the game,
about 30% of the time it immediately unpauses itself.
