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
will show and automatically dismiss the password screen
unless joypad auto polling is correct.

Also, at the title screen,
tapping B should open the main menu,
but holding it for more than a few frames
*should* automatically select "New Game".
If the HVBJOY register
does not correctly report whether auto joypad polling is active,
this behaviour may be off in either direction
(you can't tap B quickly enough to open the menu
without selecting "New Game",
or holding B will never select "New Game").

# Sufami Turbo (Japan)

Not necessarily an emulation bug, but a hazard.
Unlike other games with special hardware,
there's no value in the internal header to tell you that
a game accepts Sufami Turbo mini-cartridges.
Instead, the emulator has to guess whether
something "looks like" the Sufami Turbo base cartridge.

You could detect it by the size and hash of the ROM,
by the game's internal name (`ADD-ON BASE CASSETE`),
or by the text at the beginning of the ROM (`BANDAI SFC-ADX`).
You might be tempted
to use the four-letter game code in the extended header
to identify the cartridge,
but the basic header does not set the "has extended header" flag,
but beware:
this has the game code set to `A9PJ`,
which is shared with the game
"Bishoujo Senshi Sailor Moon SuperS - Fuwafuwa Panic".

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

# Wolverine - Adamantium Rage (USA)

This game hangs after showing the developer and publisher logos,
before it gets to the title screen,
unless auto joypad polling is correct.

Strangely,
the European version
and the Japanese version
(which is just called *Wolverine (Japan)*)
can work fine even if the USA version is broken.

# World Masters Golf (Europe)

On the main menu,
holding the D-pad moves the cursor continuously
instead of only once,
unless auto joypad polling is correct.

# Zenkoku Koukou Soccer 2 (Japan)

During a match,
holding the Start button will pause the game about two thirds of the time,
and one third of the time the game will immediately unpause itself.
If auto joypad polling is incorrect,
the game might always pause reliably,
or never pause reliably.

This is because when the game is paused,
it repeatedly re-reads the joypad registers,
and can wind up reading partially-complete data
during the auto-polling process.
