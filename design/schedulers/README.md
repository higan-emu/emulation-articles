One of the most important fundamentals for a solid emulator foundation is a
quality scheduler. When emulating multiple processors of a target system, we
need a way to keep track of where each processor is at in time relative to the
other processors, so that we can emulate them at their appropriate clock speeds.

There are two major ways to do this, and which one to choose depends on how
complex the system you are trying to emulate is.

![How time is tracked in hardware: oscillator circuit diagram and quartz crystal](/images/design/schedulers/oscillators.jpg)

Foreword: in this article, I'll refer to each emulated processor as being a
thread, specifically one that is [cooperative](/design/cooperative-threading)
or state-machine based. Thus I'll discuss below how to implement thread
schedulers. These threads can refer to any component in a system: a general
processor, a graphics chip, an audio chip, etc.

# Relative Schedulers

Relative schedulers are fast, easy, and if done right, essentially perfect at
keeping track of time. Their biggest weakness is that they only work in a 1:1
relationship between any two given threads. You can of course create as many 1:1
relationships as you wish, but eventually it will grow too complex and expensive
to be an effective tool.

A great use-case for a relative scheduler would be the Super Nintendo, and a
terrible use-case for it would be the Sega CD.

To implement a relative scheduler, one needs to hold a signed 64-bit integer
for each 1:1 thread relationship.

Let's look at the layout of the Super Nintendo: the base system is comprised of
a general processor (CPU), an audio coprocessor (SMP), a graphics chipset (PPU),
and an audio generator (DSP).

For the SNES, the DSP can only directly communicate with the SMP, and the PPU
can only directly communicate with the CPU. That gives us the following needed
64-bit integers:

* CPU <-> PPU
* CPU <-> SMP
* SMP <-> DSP

Say we want to track the relative time between the CPU, which runs at
21,477,272hz, and the SMP, which runs at 24,576,000hz. We will have a scheduler
variable named `int64 cpu_smp` for this purpose. This represents a
relative-timed clock. At power-on and reset, we set this clock variable to zero.

Whenever the counter is >= 0, we consider the CPU to be ahead of the SMP, and
whenever the counter is < 0, we consider the SMP to be ahead of the CPU.
Whenever the CPU steps by N clocks, we subtract `N * 24,576,000` from `cpu_smp`.
And whenever the SMP steps by N clocks, we add `N * 21,477,272` to `cpu_smp`.

Note: technically, a counter of zero indicates both processors are at the exact
same moment in time, but since we need to choose a thread to run, we consider
one thread to be ahead in time whenever the counter is >= 0, rather than only
when the counter is > 0.

The key detail here is that the CPU subtracts `N * SMP_frequency`, and the SMP
adds `N * CPU_frequency`. We keep relative time by stepping by a multiple of the
*other* thread's frequency.

The CPU <-> PPU happen to run off the same 21,477,272hz clock, so a slight
optimization can be made in that case: we can omit the scalar: whenever the CPU
steps by N clocks, we subtract N from `cpu_ppu`; and whenever the PPU steps by N
clocks, we add N to `cpu_ppu`.

There is an issue of overflow / underflow here, especially for the CPU <-> SMP
example, which is why we use a 64-bit signed integer. This gives us 63-bits of
usable precision in either direction, and 2^63 / 24,576,000 tells us that the
CPU can advance up to 375,299,968,947 clocks ahead of the SMP before `cpu_smp`
would underflow. That's 17,474 seconds, which seems like more than enough time
to ever worry about it. But the general idea is that you would never want to
allow the CPU to run more than that amount of time ahead of the SMP.

You can just slightly start to see the issue with relative scheduling by noting
that whenever the CPU steps by N clocks, it needs to update both `cpu_smp` and
`cpu_ppu`. The more processors you add in, the more work this becomes.

Take the Sega CD, where you have two 68K CPUs, a Z80 APU, a VDP graphics chip,
a PSG audio chip, a YM2612 FM synthesis chip, a CD drive controller, a custom
ASIC graphics scaler, and more ... almost all of which can directly communicate
with each other, and a relative scheduler turns out to be a rather bad choice.

# Absolute Schedulers

Absolute schedulers are somewhat more involved, but can be used to inspect the
current time of any one thread to any other thread, or N:N. This makes it a
great choice for the aforementioned Sega CD, among other systems.

The idea here is that each thread keeps a 64-bit unsigned integer timestamp, and
as it executes time, we increment its counter. If this counter is ahead of the
one for another thread, it is ahead in time.

Of course, these counters will eventually overflow as well, and so periodically,
you will have to check every single thread counter to find the smallest value
(that is to say, the thread that is the furthest back in time), and subtract
that value from every single thread. You will further have to ensure that no
thread can run long enough to overflow its 64-bit counter.

The problem we face with an absolute scheduler is that multiple disparate clock
frequencies, say 21,477,272hz and 24,576,000hz as for the Super Nintendo, do not
map directly into a consistent unit of time, say nanoseconds.

And so we first have to normalize all of our processor clock rates to some unit
of time, such as nanoseconds. Since real oscillators are also not 100% exact, we
don't really need absolute perfect synchronization, and can spare a slight
fractional rounding error. But we can still do much better than nanoseconds.

Also, as mentioned above, we will use a 64-bit unsigned integer, rather than
floating point for performance reasons.

The first thing we have to define is how much time the 64-bit range can express,
which will be the maximum amount of time the thread furthest in the future is
ahead of the thread furthest in the past.

I've found one second of time to be a good general choice overall. To detect
when threads need to be normalized, I use the 64th-bit for this purpose. Thus,
whenever a counter reaches 2^63, that signifies it is time to find the thread
furthest behind in time and subtract that counter from all threads.

Thus, we define a constant `Second` as 2^63 - 1.

Now that we know how many numbers can fit into a second, we can normalize each
thread's frequency to it.

Thus, we define each thread's constant `Scalar` as `Second / Frequency`.

Whenever a thread steps by N clocks, its 64-bit unsigned counter is incremented
by `N * Scalar`.

If we have 2^63 - 1 ticks per second, then we are thus able to track time at
ten times the precision of attoseconds (which is 10^18.)

And if you really want to drive home excessive overkill: 64-bit CPUs can do
128-bit math, and types such as `uint128` can be used to provide you with
2^127 precision. Of course, that's really not necessary at all.

Again, you can balance this trade however you want: give up more precision to
allow a longer maximum run-ahead time (> 1 second), or give up more run-ahead
time for better precision. But really, the above 64-bit counters and one-second
of time distance should be good enough for virtually every use case.

# Implementation

My emulator [bsnes](https://github.com/bsnes-emu/bsnes) uses a relative scheduler. You
may find its implementation on GitHub here:

https://github.com/bsnes-emu/bsnes/blob/3808e8e25fa30515fb5888c0ef5063cf415edd96/bsnes/sfc/sfc.hpp

My emulator [higan](https://github.com/higan-emu/higan) uses an absolute scheduler. You
may find its implementation on GitHub here:

https://github.com/higan-emu/higan/tree/2a110b44631f01c0c5d64fab6a782320d8fb0ec6/higan/emulator/scheduler

As you can see, the latter is considerably more complex, and for the simple case
of the Super Nintendo, it is more performant to use a relative scheduler.

higan however emulates so many systems, many of which would not scale well with
a relative scheduler, and so an absolute scheduler is used there instead.
