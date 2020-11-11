Some games work flawlessly even on wildly inaccurate emulators,
others require exact emulation of subtle behaviour
(and even then [some](../../game-bugs/snes/) still don't work).

# Nuke (PD)

Inputs do not work unless auto joypad polling is correct.

# SpellCraft - Acts of Valor (USA) (Proto)

At the title screen,
pressing B may bring up the main menu
and automatically select New Game
rather than letting you choose a different option.

This occurs if the HVBJOY registor
does not correctly report whether auto joypad polling is active.

# Super Conflict

Sends random inputs even with no buttons pressed,
unless auto joypad polling is correct.

# Super Star Wars

Start button auto-unpauses
unless auto joypad polling is correct.

# Taikyoku Igo - Goliath

Start button does not work at the title screen
unless auto joypad polling is correct.

# Tatakae Genshijin 2

Attract sequence ends early
unless auto joypad polling is correct.

# Williams Arcade's Greatest Hits

Inputs fire on their own,
or menu items sometimes skipped
unless auto joypad polling is correct.

# World Masters Golf

On the main menu,
holding the D-pad moves the cursor continuously
instead of only once,
unless auto joypad polling is correct.
