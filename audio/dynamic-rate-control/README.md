(updated: 2019-10-01)

When running a native game directly on your PC, video can be synchronized to the
vertical retrace, and entire audio tracks can be placed into memory to play on
their own. Sound effects are triggered as needed. The result is a seamless
experience.

But when it comes to emulation, this becomes much more difficult. Unlike with PC
games, we don't know the entire audio track in advance, and so we have to stream
samples as they are generated under emulation.

In this article, I'll dive into why that is, and one technique that can address
the problem.

# Cycle Timing

Emulating retro video game systems is difficult due to the sheer sizes of the
game libraries. With thousands of titles per system, you can bet that at least a
few will break if the timing is off by even a little. That is, if the emulation
runs a touch too fast or too slow.

With rare exception, in order to reach 100% compatibility, an emulator for older
systems must be cycle accurate.

This means that the original video frequency (rate of frames rendered per
second) and audio frequency (rate of samples played per second) must be exactly
the same as the original hardware: at least, as far as the emulated environment
can observe.

But in order for an emulator to produce smooth video with no tearing or
stuttering, and smooth audio with no crackling or popping, the ratios of
frequencies must be an exact match if they are static. Which is, regrettably,
just not possible, as I'll explain.

# Frequencies and Oscillators

Unfortunately, these frequencies are not simple numbers. You'll often find tech
specs that will state information such as the Super Nintendo having a 60hz video
refresh rate, and a 32KHz audio sampling rate in the NTSC region. But that is
not exactly true.

The video and audio frequencies are based on the underlying oscillator
frequencies. An oscillator can be a quartz crystal clock or a ceramic resonator.
I'll delve into the differences later.

The SNES NTSC CPU frequency is `315 / 88 * 6000000hz`, or approximately
~21.477MHz.

The SNES APU frequency is `32000 * 768hz`, or approximately ~24.576MHz.

The CPU frequency is based around the NTSC color subcarrier, which is part of
how the SNES video chipset (PPU) can render an image onto NTSC CRT monitors.

One scanline on the SNES takes 1364 clock cycles, and one frame takes 262
scanlines. So we can thus derive the SNES video refresh rate as:
`315 / 88 * 6000000 / 1364 / 262`, or 60.098477561hz. (I am simplifying a
touch by omitting one detail regarding a missing 4hz period for NTSC color
subcarrier shifting for the sake of simplicity here.)

One sample on the SNES takes 768 clock cycles, which gives us:
`24576000 / 768`, or 32000hz.

# Dynamic Video Rates

... only the SNES refresh rate isn't quite so simple: when enabling interlacing,
an extra scanline is inserted into odd frames, to provide 525 scanlines every
two fields.

So that is `315 / 88 * 6000000 / 1364 / 525 * 2`, or 59.9840042665hz.

One might suspect that means there are two refresh rates to worry about, but
unfortunately the reality is more complex: there is nothing stopping a game,
other than a lack of reason, from toggling interlacing on and off constantly
within the same second of time. This ends up producing effective video
frequencies that can range anywhere from ~59.98hz to ~60.09hz.

# Oscillator Inaccuracies

As if that's not enough ... the ugly truth to electrical engineering is that
there's really no such thing as a perfect circuit. Everything is built around
tolerances.

When it comes to quartz crystal oscillators like the SNES CPU uses, there is a
small tolerance range. So instead of being exactly ~21.477MHz as per above, the
actual rate can vary.

The age of the oscillator, its manufacturing run, and even the current
temperature can slightly affect the oscillator rate. That's right, even while
your SNES console is running, if you were to connect an oscilliscope and
monitor the output frequency, you would see it slightly fluctuating over time.

The SNES APU oscillator is a ceramic resonator. These are generally considered
to be less expensive, and have even less accuracy. That is to say, more
variance.

In fact, most observations place SNES APU oscillators to be closer to
`32040 * 768`, or ~24.607MHz, in practice.

# Host Computer Inaccuracies

Yep, the SNES frequencies are not quite exact, and that means your PC
frequencies are also not exacting.

When you set your PC monitor to 60hz and your audio output to 48KHz, that's not
*exactly* what you're getting. It may be something closer to 60.1hz and
48.03KHz.

And even if we tried to measure it, it would just fluctuate over time.

# Synchronization

When emulating a system, generally speaking we're going to be able to run the
emulated system much faster than the original hardware. We have to cap the speed
somehow.

# Synchronizing to Video

One method is to rely on the emulated system and host system video refresh rates
being mostly the same. So if your SNES is 60.09hz, and your PC monitor is 60hz,
by synchronizing to the vertical blanking period of the monitor, we can ensure
that the video is always smooth, even during scrolling segments, with no tearing
or stuttering.

The emulated system will actually end up running 0.15% too slow, but such a
small difference is unlikely to be observed.

Unfortunately, this leaves a serious problem for audio: if the ratio of video
samples to audio samples is not exact, one of two things will happen.

If the video ratio is too high, the audio buffer will slowly deplete until it is
empty. The sound card will need audio samples to play, and none will be
available.

Depending on your audio API, this will either cause the audio buffer to loop
back to the beginning of the sample buffer (eg DirectSound), or the sound driver
to send silence to the sound card (eg XAudio2.) Although the latter is
preferable, the result is the same: the audio will pop and sound quite terrible.
Once this happens, the buffer will recover and sound will become clear again.
But only for a time.

If the SNES refresh rate to PC refresh rate is off by 60.09 -> 60.00hz, that
means that roughly once every ten seconds, the audio will pop.

if the video ratio is too low, the audio buffer will fill up completely. We
cannot simply grow the audio buffer forever, because the larger the audio
buffer, the more input lag there will be between what we see onscreen and what
we hear from our speakers. And so we are forced to either write over the front
of the buffer, or drop the sample completely. The latter is slightly better, but
again, a loud pop or click will be heard every time this happens.

# Synchronizing to Audio

Another method is to synchronize the emulator to audio instead: say we know the
SNES audio is approximately 32KHz, and the PC sound card's native rate is
approximately 48KHz. If we run the audio through a simple resampler, we can
generate 3 output samples per 2 input samples, or 3:2 ratio.

We run the emulation, and fill the audio buffer. Any time the audio buffer is
full, we stop the emulation and wait until space is available to add more
samples.

This results in perfectly clear audio, but the reverse problem now happens to
our video.

If the audio ratio is too high, we will hit a vertical blanking period but have
no frame to display. We can either render frames even during active display,
which will cause a creeping tearing bar down the screen, or we can skip a frame,
resulting in stuttering that is very evident during scrolling scenes.

If the audio ratio is too low, we will end up having two frames ready at the
next vertical blanking period, and again we will have to draw early, or drop a
frame, resulting in either tearing or stuttering.

# Synchronizing to Both Video and Audio

You might think to synchronize to both, but instead of fixing both problems,
this ends up causing both problems: your video will stutter or tear, and your
audio will pop and crackle. It's the worst of both worlds.

When either video or audio blocks the emulation, it will stall out the other.

# Static Rate Control

One technique that's somewhat effective is static rate control. In this mode, we
do actually synchronize to both video and audio.

We know that the exact oscillator rates of both the emulated system and the host
system for both video and audio tend to fluctuate, and so trying to determine an
ideal state rate is not possible. But we can provide the user with a slider that
allows them to fine-tune the ratio.

So imagine your emulator has a slider for the audio resampler that lets the user
adjust the audio ratio from 1.9:3 all the way to 2.1:3 ratio.

What will happen when they move this slider is that one direction will start
causing more and more video stuttering, and the other direction will cause more
and more audio stuttering. But by carefully aiming for the middle ground, you
can end up with a situation where the stuttering only occurs very infrequently.

With practice, the best I've managed was about ten minutes of perfectly
synchronized video and audio at a time, per 20ms stutter, with this approach.

This is however still not perfect, and it's very difficult to explain all of the
above to end users running your emulator.

# Dynamic Rate Control

But what if we could adjust the audio resampling ratio in real-time? Yes, that
is the gist of dynamic rate control.

In this mode, we will synchronize only to the video. The goal with dynamic rate
control is to keep the audio buffer approximately half-full (or half-empty, if
you're a pessimist), at all times.

By utilizing audio APIs to query the remaining samples left in the buffer, or
the reverse, how much free space is left in the buffer, we can watch the buffer
every time a few samples have been output. If the buffer starts emptying, then
we can decrease the audio ratio so that more samples are generated, and the
buffer begins to fill back up. This will inevitably end up going too far, and
the buffer will start filling beyond the halfway point. And so we compensate in
the reverse and increase the ratio so that the buffer begins to deplete.

By constantly making micro adjustments to audio resampler ratio, we keep the
buffer in a state where it will hopefully never fill and never empty.

Now as above, if the SNES is supposed to run at 60.09hz, and your PC monitor
runs at 60hz, this will still cause the emulation to be running 0.15% too slow,
but at least now both the video and audio will always be perfectly synchronized.

# Pitch Distortion

By adjusting the audio resampling ratio, we do actually alter the pitch. And so
it's very important that we never adjust the pitch *too* much in any one step,
or the audio will sound extreemly unpleasant.

# Implementation

Dynamic rate control isn't a difficult concept to understand, but it can be a
touch tricky to implement, and so I'll walk you through that with some code now.

The code will be slightly abridged here, but the full source code can be found
in my [GitHub repository](https://github.com/higan-emu/higan/tree/master/ruby/audio) for all of this code.

Consider the code in this article public domain. This code assumes
floating-point input samples and produces 16-bit signed output samples, which is
more a design choice in my emulators. You may adapt the code to your needs.

## Cubic Resampler

The first thing we need is a resampler that takes both an input and an output
frequency, to perform eg the ~2:3 resampling ratio we talked about before. I'll
delve into audio DSP in a later article, but for now, here's a simple cubic
resampler. This code is just rather standard interpolation, it's not too
important to delve into exactly how it works here.

```
auto Cubic::reset(double inputFrequency, double outputFrequency, uint queueSize) -> void {
    this->inputFrequency = inputFrequency;
    this->outputFrequency = outputFrequency ? outputFrequency : this->inputFrequency;

    ratio = inputFrequency / outputFrequency;
    fraction = 0.0;
    for(auto& sample : history) sample = 0.0;
    samples.resize(queueSize ? queueSize : this->outputFrequency * 0.02);  //default to 20ms max queue size
}

auto Cubic::setInputFrequency(double inputFrequency) -> void {
    this->inputFrequency = inputFrequency;
    ratio = inputFrequency / outputFrequency;
}

auto Cubic::pending() const -> bool {
    return samples.pending();
}

auto Cubic::read() -> double {
    return samples.read();
}

auto Cubic::write(double sample) -> void {
    auto& mu = fraction;
    auto& s = history;

    s[0] = s[1];
    s[1] = s[2];
    s[2] = s[3];
    s[3] = sample;

    while(mu <= 1.0) {
        double A = s[3] - s[2] - s[0] + s[1];
        double B = s[0] - s[1] - A;
        double C = s[2] - s[0];
        double D = s[1];

        samples.write(A * mu * mu * mu + B * mu * mu + C * mu + D);
        mu += ratio;
    }

    mu -= 1.0;
}
```

## Dynamic Rate Controller

We want to support several audio APIs (eg waveOut, DirectSound, OSS, etc), and
the basic method of controlling the resampling ratio can be abstracted to a base
class:

```
auto Audio::output(const double samples[]) -> void {
    if(!instance->dynamic) return instance->output(samples);

    auto maxDelta = 0.005;
    double fillLevel = instance->level();
    double dynamicFrequency = ((1.0 - maxDelta) + 2.0 * fillLevel * maxDelta) * instance->frequency;
    for(auto& resampler : resamplers) {
        resampler.setInputFrequency(dynamicFrequency);
        resampler.write(*samples++);
    }

    while(resamplers.first().pending()) {
        double samples[instance->channels];
        for(uint n : range(instance->channels)) samples[n] = resamplers[n].read();
        instance->output(samples);
    }
}
```

Here, we call output() for every frame (pair of samples) generated from within
our emulation. maxDelta controls the maximum pitch distortion.

Calling instance->level() is how we query the current buffer fill level for any
given audio driver. And calling instance->output() will actually send the sample
to the driver to output to the speakers.

## OSS (*nix)

OSS is the tried and true classic audio interface for *nix systems. While rather
deprecated on Linux, it is available through an emulation layer there. It also
works very well under the BSDs.

```
auto AudioOSS::level() -> double override {
    audio_buf_info info;
    ioctl(_fd, SNDCTL_DSP_GETOSPACE, &info);
    return (double)(_bufferSize - info.bytes) / _bufferSize;
}

auto AudioOSS::output(const double samples[]) -> void override {
    for(uint n : range(self.channels)) {
        buffer.write(sclamp<16>(samples[n] * 32767.0));
        if(buffer.full()) {
            write(_fd, buffer.data(), buffer.capacity<uint8_t>());
            buffer.flush();
        }
    }
}
```

Here, SNDCTL_DSP_GETOSPACE lets us determine the buffer fill position.

## waveOut (Windows)

waveOut is perhaps the oldest audio API on Windows, and yet it turns out to have
some of the most reliable buffer polling of any API on the platform. Go figure.

```
auto CALLBACK waveOutCallback(HWAVEOUT handle, UINT message, DWORD_PTR userData, DWORD_PTR, DWORD_PTR) -> void {
    auto instance = (AudioWaveOut*)userData;
    if(instance->blockQueue > 0) InterlockedDecrement(&instance->blockQueue);
}

auto AudioWaveOut::level() -> double override {
    return (double)((blockQueue * frameCount) + frameIndex) / (blockCount * frameCount);
}

auto AudioWaveOut::output(const double samples[]) -> void override {
    uint16_t lsample = sclamp<16>(samples[0] * 32767.0);  //ensure value is between -32768 and +32767
    uint16_t rsample = sclamp<16>(samples[1] * 32767.0);

    auto block = (uint32_t*)headers[blockIndex].lpData;
    block[frameIndex] = lsample << 0 | rsample << 16;

    if(++frameIndex >= frameCount) {
        frameIndex = 0;
        while(waveOutWrite(handle, &headers[blockIndex], sizeof(WAVEHDR)) == WAVERR_STILLPLAYING);
        InterlockedIncrement(&blockQueue);
        if(++blockIndex >= blockCount) {
            blockIndex = 0;
        }
    }
}
```

Determining buffer fill levels with waveOut is done by taking advantage of its
buffer queuing system. We push several small blocks worth of audio, and then we
can read how many blocks are still in the queue. Since this is multi-threaded,
we do need to use InterlockedIncrement/Decrement here, but that's not hard.

Querying entire blocks instead of individual samples isn't very precise, and so
we make up for that by creating more small blocks instead of less large blocks.

The exact ratio honestly comes down to experimentation. In my own case, I find
32 small blocks to be a sufficient queue size for dynamic rate control to work
well.

# Adaptive Sync

There is an even better technique which can emulate a given system at its exact
original speed, and which doesn't need any rate control whatsoever: it is known
as adaptive sync, which is where instead of the video card outputting frames at
a constant ratio, your emulator takes control of when to refresh the screen.

This approach has several downsides: few people have adaptive sync monitors,
even fewer have monitors that can go above 60hz (which is needed for systems
like the 75hz Bandai WonderSwan), the technique doesn't work all that well in
windowed mode (although it *can*), and it also requires very low-latency audio
drivers (WASAPI, JACK, etc) which are much harder to get working everywhere.

However, it has one upside: dynamic rate control only works when the emulated
and host systems have nearly identical video refresh rates, and especially in
windowed mode, it's not practical for an emulator to change the PC monitor's
refresh rate.

But this article is about dynamic rate control. I will talk more about adaptive
sync in a future article.

What I will say in closing is that emulator developers should really strive to
implement both dynamic rate control and adaptive sync, to cover all bases.

# Credits

I would like to give credit to BearOso for helping me with computing the audio
buffer level() functions above.

I am not 100% certain on who pioneered this idea, but currently the earliest
known example of dynamic rate control in an emulator is nemulator, which added
support for this in 2006.
