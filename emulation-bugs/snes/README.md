Some games work flawlessly even on wildly inaccurate emulators,
others require exact emulation of subtle behaviour
(and even then [some](../../game-bugs/snes/) still don't work).

# Nuke (PD)

Inputs do not work unless auto joypad polling is correct.

# Secret of Mana (USA)

When riding the dragon Flammie
(in the initial mid-altitude over-the-shoulder view,
not the low-altitude overhead view
or the whole-planet view),
the Mode7 background will flicker black
unless joypad auto polling is correct.

# SpellCraft - Acts of Valor (USA) (Proto)

After starting a new game,
walking to the red "Get Password" orb and pressing B
will automatically dismiss the password screen
unless joypad auto polling is correct.

Also, at the title screen,
tapping B should open the main menu,
but holding it for more than a few frames
*should* automatically select "New Game".
If the HVBJOY registor
does not correctly report whether auto joypad polling is active,
this behaviour may be off in either direction
(you can't tap B quickly enough to open the menu
without selecting "New Game",
or holding B will never select "New Game").

# Super Conflict (USA)

Sends random inputs even with no buttons pressed,
unless auto joypad polling is correct.

# Super Star Wars (USA)

Start button auto-unpauses
unless auto joypad polling is correct.

# Taikyoku Igo - Goliath (Japan)

Skips the intro animation,
jumps straight to the title screen,
and won't let you start the game
unless auto joypad polling is correct.

# Tatakae Genshijin 2 (Japan) (En)

Attract sequence ends early
unless auto joypad polling is correct.

# Williams Arcade's Greatest Hits (USA)

Inputs fire on their own,
or menu items sometimes skipped
unless auto joypad polling is correct.

# World Masters Golf (Europe)

On the main menu,
holding the D-pad moves the cursor continuously
instead of only once,
unless auto joypad polling is correct.
