There are many sources of latency involved in emulating game consoles. In this
article, I'll go over the one input latency source we have direct control over
as emulator authors, and propose my method for minimizing it.

# Input Polling

An emulated game system typically reads from controllers once per emulated
frame during the non-maskable vertical blank interrupt.

Technically, games can poll controllers at any time, and as many times as they
want, but 99.9% of games will follow the above strategy.

Usually, inputs will be available from a memory-mapped I/O register that is
polled. Emulators then must convert the emulated mapping to a physical keyboard,
mouse, or gamepad, and return the current button state to be returned from
reading said I/O register.

# Typical Workflow

Most emulators will be designed to poll the real hardware devices once per
frame, in a run loop that looks something like this:

```cpp
void Program::run() {
        while(stopped() == false) {
        hardware.pollInputs();
        emulator.runFrame();
        video.drawFrame();
    }
}
```

In this case, `hardware.pollInputs()` will query DirectInput, XInput2, SDL, or
whatever hardware input APIs are available, and cache the results for later use:

```cpp
void Hardware::pollInputs() {
    keyStates = directInput.pollKeyboard();
}
```

Next, `emulator.runFrame()` will run the emulator continuously until one frame
of video data is ready, at which point the function will return control back to
the program.

Inside the emulation core, when the memory-mapped I/O register for gamepad
states is read, the cached states are returned:

```cpp
uint8_t Emulator::pollGamepad() {
        uint8_t data = 0;
    data |= program.readInput(GAMEPAD_UP  ) << 0;
    data |= program.readInput(GAMEPAD_DOWN) << 1;
    ...
    return data;
}
```

This is done by the program translating the emulated inputs to mapped hardware
inputs:

```cpp
bool Program::readInput(uint inputID) {
    if(inputID == GAMEPAD_UP  ) return hardware.keyStates[KEY_UP];
    if(inputID == GAMEPAD_DOWN) return hardware.keyStates[KEY_DOWN];
    ...
}
```

Finally, `video.drawFrame()` will take the emulated video frame and output it
to the onscreen window for the program, using Direct3D, OpenGL, SDL, etc:

```cpp
void Video::drawFrame() {
    direct3D.draw(program.window, emulator.frame, emulator.width, emulator.height);
}
```

This is how the multi-emulator frontend RetroArch is written.

# Missed Frames

For the purposes of this article, let's assume we are talking about the Super
Nintendo, which has 262 scanlines per frame. Let's also declare "V" to mean the
vertical scanline the emulator is currently generating, from V = 0-261.

If `emulator.runFrame()` emulates an entire frame, including the vertical
blanking period, then we end up at V=0 when `hardware.pollInputs()` is called.
This state remains until V=225 where it is read by the game, so by the time we
actually use the polled results, almost an entire frame, or 16ms, has passed.

(Tangent: if the emulator is extremely performant and synchronizes itself to
video, it could reach this state faster, and then sleep the program until the
host machine reaches its own vertical blanking period.
[Synchronizing an emulator to audio](/audio/dynamic-rate-control),
or simply being a demanding emulator, would negate this benefit.)

# Avoiding Missed Frames

Games typically draw the screen, and then enter into a vertical blanking period.
Games typically poll the emulated controllers during vertical blanking.

Again assuming this is the Super Nintendo, the screen is rendered between
V = 1-224 for NTSC mode, and V = 1-239 for PAL (overscan) mode.

One strategy is for the emulator to exit right after the screen has rendered,
but before the inputs are polled. In other words, right at the start of the
emulated vertical blanking period.

If `emulator.runFrame()` returns at V=225 (for NTSC) or V=240 (for PAL), then
we will poll the host machine inputs right before returning to the emulated
machine's vertical blank handler that then polls the inputs.

This is how lag-fix patches are made for RetroArch, which seek to remove this
potential one-frame lag penalty.

# Complications

Overscan is actually a setting that tells the SNES to render an additional
15 scanlines, which can be readily seen on a PAL display. However, there are
NTSC games that do use overscan, and PAL games that do not use overscan.

In fact, it's even possible, if pathological, to toggle the overscan setting
during the vertical blanking period. The Titan Overdrive 2 demoscene software
for the Sega Genesis abuses the Genesis' graphics chip to eke out additional
scanlines beyond what the original hardware was capable of even.

But even for the SNES, if we reach V=225 with overscan disabled, that doesn't
guarantee the entire frame has been rendered: a game may turn on overscan and
happily start drawing more scanlines. That's a big problem for trying to exit
`emulator.runFrame()` at V=225.

Also consider that games may not necessarily choose to poll the controllers at
exactly V=225. Some games may choose to poll inputs at V=220, before the
screen ends, or at V=261, right at the end of the vertical blanking routine.

A pathological game might even choose to poll inputs whenever it has available
cycles to do it, and the polling location may change every single frame as a
result.

The above optimization is too coarse. We can do better.

# Polling Every Scanline

The PC Engine emulator Ootake tries to improve upon this by polling the real
hardware inputs once every emulated scanline. That certainly works, but that's
262 DirectInput polling events per emulated frame. At a 60hz refresh rate, that
works out to 15,720 calls per second to the DirectInput API.

This is just wasteful as even the fastest USB devices only poll 1,000 times per
second. And usually it's only 100 times in default OS configurations.

# Hardware Polling Overhead

Calling `hardware.pollInputs()` is expensive: we have to query hardware APIs
that likely require kernel transitions, get entire keyboard states, map those
states to our emulated inputs, etc.

This is why emulators try to only do this once per frame. If not for this
overhead, the easy solution would just be for `Program::readInput()` to
call out to `hardware.pollInputs()` itself.

But there is a simple solution to this.

# Proposal: Just-in-Time Polling

For lack of a better term, I'll just call this JIT polling.

By keeping a timestamp of the last time the host hardware inputs were polled, we
can short-circuit doing the actual polling if called too frequently.

This new design would look something like this:

```cpp
void Hardware::pollInputs() {
    static uint64_t lastPollTime = 0;
    uint64_t thisPollTime = getHostMachineCurrentTimestampInMilliseconds();
    if(thisPollTime - lastPollTime >= 5) {  //latency timeout: 5 milliseconds
        keyStates = directInput.pollKeyboard();
        lastPollTime = thisPollTime;
    }
}

void Program::run() {
    while(stopped() == false) {
    //we no longer have to call hardware.pollInputs() here
        emulator.runFrame();
        video.drawFrame();
    }
}

bool Program::readInput(uint inputID) {
    //we call hardware.pollInputs() here instead
    hardware.pollInputs();
    if(inputID == GAMEPAD_UP  ) return hardware.keyStates[KEY_UP];
    if(inputID == GAMEPAD_DOWN) return hardware.keyStates[KEY_DOWN];
    ...
}
```

What the above code does is remove the need to care about when
`emulator.runFrame()` returns: it simply doesn't matter anymore.

Whenever the emulated system tries to poll the inputs, we poll the host machine
inputs at that time. And because we have the 5 millisecond timeout, that means
`Emulator::pollGamepad()`'s second call to `Program::readInput()` to get
the state of `GAMEPAD_DOWN` will not call `hardware.pollInputs()` a second
time.

The hardware is polled immediately before the inputs are read, and then all of
the inputs are almost always read together in one cluster by the emulation core,
and so `hardware.pollInputs()` is only invoked once per frame.

Now that we've polled the hardware, the emulator continues running for another
frame, and so another 16-20 milliseconds have passed, and so once again, the
host hardware is polled immediately, no matter when the emulated game has
decided to poll the inputs, presuming a typical game that polls inputs once per
frame.

And in the event you run into a pathological game that does something like
continually poll the input every scanline, we have guaranteed that the maximum
latency is 5 milliseconds on the host machine.

In 99.9% of cases, JIT polling will only poll DirectInput 60 times per second,
the same as the original flawed technique first discussed in this article.

In the pathological case, the 5 millisecond timeout sets an upper-bound on the
maximum overhead, and also the maximum input latency: 5 milliseconds, or 200
DirectInput calls per second.

This is how [bsnes](https://github.com/bsnes-emu/bsnes) and [higan](https://github.com/higan-emu/higan) emulate
input polling. And this is why I did not merge the RetroArch lag-fix patch into
bsnes in the past: it was needed for RetroArch's design, but not for bsnes'
design. And it would have interfered with my emulation of SNES overscan being
possible to toggle during vertical blanking periods.

# Variadic JIT Polling

A cute side-effect of JIT polling is that the 5 millisecond delay can be turned
into a user-configurable variable. Set as low as 1 millisecond, you can keep up
with the best 1000hz USB polling gamepads and drivers on the market. And if you
set the value higher than it takes to render one frame, it becomes a lag
simulator! I have no idea why anyone sane would want to do this, but ... input
latency is usually a rather abstract concept to users: it's not directly
measurable without high-speed 240fps+ cameras and careful analysis of
side-by-side video.

JIT polling with an adjustable latency timeout allows users to simulate what
it's like to have more input lag. By setting it to 50ms, they introduce
approximately two additional frames of input lag, and they can see by direct
hands-on experience how two additional frames of latency affects the playability
and feel of the game.

I suppose it could also make for a fun party trick: an easy way to greatly
increase the difficulty of a game could be to raise the input latency. An easy
handicap in a fighting game would be to have a larger latency period for one
player than the other. Practical or useful? Probably not so much.

# Closing

I believe that by applying this technique to other emulators such as RetroArch,
we can effectively lag-fix *every* emulation core in one fell swoop, without
requiring any patches to upstream emulators.

I further believe that even already lagfixed emulators have the potential to
receive very slight further input latency reductions (as in the case of my
hypothetical rare game that might poll inputs at the *end* of vertical blank
rather than at the start.)

I'm not claiming to have invented this technique: there are thousands of
emulators out there. I'm only stating that at the time of writing, I'm not aware
of this being done elsewhere, and I think it would help emulators if this
technique were to become more widespread.

I am also not claiming it's a particular stroke of brilliance to come up with:
it was a rather simple idea that worked well for me.

I am not asking for credit for this idea. Consider it public domain.

And most importantly of all, I am not criticizing anyone else's approaches. For
many years, my own emulators used the first approach as well. Lately there's
been a huge push in the emulator development community to mitigate latency, and
I couldn't be happier about this. I'll have more to say on this topic in the
future, such as an article on a very neat time-shifting latency reduction
implemented in RetroArch known as run-ahead. Stay tuned for that.

Emulation development isn't a competition, and everyone benefits from sharing
ideas and improving things. That's all I seek to do with this article.

Thanks for reading, I hope this will be of some use to folks out there.
