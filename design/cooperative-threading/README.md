(updated 2019-10-17)

Perhaps the largest challenge to writing an emulator is grappling with
programming code being inherently serial, whereas the emulated systems often
contain many, sometimes even dozens, of parallel processes all running at the
same time.

In fact, most of the overhead of writing a retro video game system emulator is
managing the synchronization between components in this environment.

In this article, I'll go into the approach I chose for my emulators: cooperative
multi-threading. Please note that this is but one of several approaches, and I
am not here to advocate for one or the other. I will do my best to present an
unbiased look at its pros and cons.

# Synchronization Overview

What we are trying to accomplish here is synchronization between emulated
components.

Let's say we are emulating the base Super Nintendo, which has four major
components: the general-purpose CPU, the audio APU, the video PPU, and the sound
output DSP. Note that I'm simplifying things a bit here (eg there are two PPU
processors) for the sake of example.

Each of these chips run at clock rates of around ~20MHz.

Our emulator needs to run each chip in serial, or one at a time; whereas the
real SNES would run all of these chips in parallel, or all at the same time.

After running one chip for a certain amount of time, we synchronize the other
chips by running them for a period of time in order to catch up to the first
chip.

The more accurate you want your emulation to be, the more fine-grained you want
this synchronization to be.

You could execute one CPU opcode at a time, render one scanline at a time, and
produce one audio sample at a time. And you would have a very fast SNES emulator
which was about as accurate as those released in the late 1990s.

Or you could execute one CPU opcode cycle at a time, render one pixel at a time,
and split audio sample generation into 32 distinct cycle stages. And you would
likely have an emulator as demanding as bsnes.

The more fine-grained your emulation becomes, the more complex it gets to switch
from one emulated component to another. Imagine you are emulating a CPU
instruction, and after every cycle you need to switch to the video renderer to
draw just one pixel and then return back to the CPU. That's a massive increase
in technical complexity, which is what cooperative threading can help eliminate.

# Preemptive Threading

First we must address the obvious: modern processors have many cores, and even
in the rare event of single-core systems, modern operating systems provide
threading facilities to allow multiple tasks to run in parallel in the form of
threads. Can't we use these to write emulators?

The answer is, unfortunately, no.

Emulating the SNES accurately involves tens of millions of synchronizations, or
context switches, per emulated second.

If your CPU only has one actual core, all of your threads will technically be
running in serial anyway, and the OS will be performing context switches in
kernel mode. Userland <> kernel mode transitions are *extremely* painful for
CPUs, and in the best case scenario, you might manage to do this a few hundred
thousand times per second. Presuming you weren't emulating anything at all.

In the case of multi-core CPUs, things do get a bit better, but whether using a
single core or multiple cores, you have to grapple with the fact that your PC is
running at a different clock rate (let's say 3GHz) than the SNES (let's say
20MHz.) The way you'd keep components in sync here would be to add locks, or
mutexes, or semaphores, in order to make one thread wait for other threads after
each slice of time has passed in each emulated thread.

These locks are immensely painful as well. Even if you bring them down to atomic
instructions such as intelocked increment and interlocked decrement, you're
still contending with modern CPUs being massively parallelized with huge
pipelines and slow communication links between each CPU core.

Not to mention in a real, non-hypothetical scenario, it's trivial to end up with
16+ emulated threads (such as emulating a Sega CD 32X machine), and demanding
that many cores to run an emulator would be a tough sell in 2019.

# State Machines

The traditional approach to retro emulator development is the use of state
machines. As an example, here's a greatly simplified example of how one might
split up a cycle-based CPU interpreter using state machines:

```cpp
void CPU::executeInstructionCycle() {
    cycle++;
    if(cycle == 1) {
        opcode = readMemory(PC++);
        return;
    }
    if(FlagM)
    switch(opcode) {  //8-bit accumulator instructions
    case 0xb9:
        switch(cycle) {
        case 2:
            address = readMemory(PC++);
            return;
        case 3:
            address = readMemory(PC++) | address << 8;
            return;
        case 4:
            //possible penalty cycle when crossing 8-bit page boundaries:
            if(address >> 8 != address + Y >> 8) {
                return;
            }
            cycle++;  //cycle 4 not needed; fall through to cycle 5
        case 5:
            A = readMemory(address + Y);
            cycle = 0;  //end of instruction; start a new instruction next time
            return;
        }
    }
}
```

Splitting up a video renderer into individual pixel timeslices, or an audio
generator into individual cycle timeslices, each has their own complexities as
well.

Now let me add yet another horror: on the SNES, the above is not enough. Each
CPU cycle consists of 6 - 12 master clock cycles, where the address bus is set
to a given address, and after a certain amount of time, other chips will
acknowledge the data on the bus. This is known as bus hold delays.

Basically, the CPU reading from the PPU cannot expect the PPU to respond with
data instantaneously: it takes time. And if the CPU is writing to the PPU, the
PPU needs some time as well.

So you would need to add a third-level state machine for every call to
readMemory() above in order to split the reads into clock cycles:

```cpp
void CPU::executeInstructionCycle() {
    if(cycle == 1) {
        opcode = readMemory(PC++);
        cycle = 2;
        return;
    }
    if(FlagM)
    switch(opcode) {  //8-bit accumulator instructions
    case 0xb9:
        switch(cycle) {
        case 2:
            switch(subcycle) {
            case 1:
                subcycle = 2;
                return;
            case 2:
                address = readMemory(PC++);
                subcycle = 1;
                return;
            }
        case 3:
            switch(subcycle) {
            case 1:
                subcycle = 2;
                return;
            case 2:
                address = readMemory(PC++) | address << 8;
                subcycle = 1;
                return;
            }
        case 4:
            //possible penalty cycle when crossing 8-bit page boundaries:
            if(address >> 8 != address + Y >> 8) {
                return;
            }
            cycle++;  //cycle 4 not needed; fall through to cycle 5
        case 5:
            switch(subcycle) {
            case 1:
                subcycle = 2;
                return;
            case 2:
                A = readMemory(address + Y);
                subcycle = 1;
                cycle = 0;  //end of instruction; start a new instruction next time
                return;
            }
        }
    }
}
```

And keep in mind, I'm still greatly simplifying things here for the purpose of
illustration.

# Cooperative Threading

Enter cooperative threading: the idea is similar to preemptive threading, but
instead of the kernel handling the threads, the userland process handles them
instead.

Cooperative threads all run in serial: that is, only one runs at a time. When
needed, a context switch will swap to another thread, which will resume exactly
where it last left off.

Unlike coroutines, each cooperative thread has its own stack frame. This means
that one thread can be several layers of function calls deep inside of a stack
frame.

The state machine code you saw above is now *implicit*: it's part of the
stack frame.

Thus, if we were to rewrite the code above, it would now look like this:

```cpp
void CPU::executeInstruction() {
    opcode = readMemory(PC++);
    if(FlagM)
    switch(opcode) {  //8-bit accumulator instructions
    case 0xb9:
        address = readMemory(PC++);
        address = readMemory(PC++) | address << 8;
        if(address >> 8 != address + Y >> 8) wait(6);
        A = readMemory(address + Y);
    }
}
```

Yes, it's really that dramatic of a difference.

How does it work? For that we look inside the readMemory() function:

```cpp
uint8_t CPU::readMemory(uint16_t address) {
    wait(2);
    uint8_t data = memory[address];
    wait(4);
    return data;
}
```

The trick is the call to wait():

```cpp
void CPU::wait(uint clockCycles) {
    apuCounter += clockCycles;
    while(apuCounter > 0) yield(apu);
}
```

wait() just consumes clock cycles, and every time it does, if the CPU gets into
a state where it's ahead of the APU, it yields execution to the APU, which then
causes the APU to resume where it last left off. Once the APU has caught up and
is now ahead of the CPU, it yields back to the CPU, and now the CPU resumes
immediately after its last yield() point.

If you think of traditional programming functions as calls, and return
statements as returning back to the call site, then you can think of cooperative
threading as jumps. Instead of returning back, the APU jumps back to the CPU.

Technically, this is referred to as symmetric cooperative threads. There are
also asymmetric cooperative threads that function like the typical call and
return style, but they're not suitable in my opinion to emulation where many
threads all run in parallel: the CPU may jump to the APU, and then the APU may
jump to the PPU rather than back to the CPU.

# Cooperative Threading Optimizations

One thing you might have noticed: why are we always synchronizing the CPU to eg
the APU after every clock cycle? What if the CPU was just reading from its own
internal memory, which the APU has no access to?

When it comes to a state machine, it's an all-or-nothing approach: you have to
split your emulated processors into the smallest timeslices possible and step by
them.

Your main emulator scheduler could see that the CPU is not trying to access the
APU on a given cycle, and run it more cycles before starting to run the APU a
bit, but in practice you're already paying the overhead through the state
machine anyway.

But cooperative threads present an opportunity for an optimization that I call
just-in-time synchronization. Let's flesh out the earlier example more:

```cpp
uint8_t CPU::readMemory(uint16_t address) {
    wait(2);
    if(address >= 0x2140 && address <= 0x2143) {
        //this read is going to access shared memory with the APU.
        //in other words, the APU might change this value before we read it.
        //as such, we *must* catch up the APU to the CPU here:
        while(apuCounter > 0) yield(apu);
    }
    uint8_t data = memory[address];
    wait(4);
    return data;
}

void CPU::wait(uint clockCycles) {
    apuCounter += clockCycles;
}
```

What we've done here is made it so the CPU will keep on running until it tries
to read from a shared memory region with the APU. Only then will it catch up the
APU to the CPU before performing the read.

If either the CPU doesn't need to synchronize to the APU often, *or vice
versa*, then the number of context switches per second drops *dramatically*.

In the case of the SNES, my emulator [bsnes](https://github.com/bsnes-emu/bsnes) only has
to synchronize the CPU and APU a few thousand times a second, instead of
millions of times a second.

# Cooperative Threading Limitations

But what happens if you have two CPUs that share all of the memory address space
with one another? Then unfortunately you wouldn't be able to know if the APU
might modify the memory before the CPU (that's currently ahead) tries to read
from it, and so you'd always have to synchronize.

There are tricks to this, such as implementing a rollback mechanism as the
Exodus emulator by Nemesis does, but you begin to lose the code simplification
benefits of this approach in that case.

In the worst case, cooperative threading reverts to the same level of overhead
as a state machine would.

# Pipeline Stalls

In fact, in the worst case, cooperative threading can be somewhat slower than
the state machine approach.

Modern CPUs love to do many operations out-of-order, and build deep pipelines of
queued instructions.

When cooperative threading changes the stack pointer, it tends to throw them
into a panic.

As an analogy, say you have an assembly line. Along the line, the product is
slowly assembled. Now say that in the middle of assembling a steady stream of
products, an order comes in for a different products, and it cannot wait, but
you only had one assembly line! You would have to stop the assembly line, take
all the items off, and start on the new products. Now imagine you had to
constantly shift back and forth between several products, tens of millions of
times a second, and you start to understand why cycle-accurate retro gaming
emulation is so demanding.

Now to be fair, state machines also suffer from this very same problem: it is
still like grinding gears to a halt during operation to switch between one area
of instructions (CPU emulation) and another (APU emulation): you're burning out
your L1 instruction cache, etc.

But the need for cooperative threading to modify the stack pointer is a real
penalty. In my admittedly non-scientific personal observations, if you can
emulate a processor using a single-level state machine, it tends to run faster
than a cooperative-threaded approach by a good margin. Whereas when you get into
two (or even three or more) levels of state machines, such as the CPU opcode
example abaove, the cooperative threading model comes out as the clear winner on
performance. And if you are able to perform the just-in-time synchronization I
previously discussed, there's no contest.

# Optimal Design

Thus, if your goal is maximum efficiency, an emulator should probably be
designed to use cooperative threading where it works best, and state machines
where it does not.

Me, I'm really big on consistency, and so for my
[higan emulator](https://github.com/higan-emu/higan), I chose to emulate every processor
using cooperative threads. I knowingly pay a performance penalty to ensure
consistency and code simplicity throughout the codebase. But of course, you
don't *have* to do that.

# Serialization

And now the elephant in the room: probably the biggest concern to emulator
developers with cooperative threading is serialization: the act of saving and
loading save states.

Save states are not only useful to save and restore your progress at any point
(great for cheating a bit at those unforgiving games, too), they are also the
building blocks for features like real-time rewind and run-ahead input latency
reduction (each of which warrants its own separate article.)

When using state machines, serialization just means saving and restoring all of
the variables used for it: opcode, cycle, and subcycle in the earlier example.

But when using cooperative threading, that information is, once again, implicit:
it's stored on the individual stacks in the form of call frames and CPU context
registers.

Implementing serialization with cooperative threading is possible, but it's a
complicated topic, for which I have penned a separate article devoted to the
topic. You can read that article here:

[Cooperative Threading - Serialization](/design/cooperative-serialization)

# Code

So if you've read this far and you're still considering giving cooperative
threading a go, you'll need an implementation of it for your chosen language.

C++20 has proposed a coroutines extension, but since as of the time of writing
this article, I don't have a working, mature implementation of it, I cannot say
if it will be useful in emulation or not. Probably the biggest limitation for it
is that C++20 coroutines are stackless, which vastly limits their potential for
anything of even moderate complexity.

Alternatively, I've developed my own cooperative threading library in C89 code,
which can thusly be used in C++ code as well, which I call
[libco](https://github.com/higan-emu/libco). You can browse its
[source code here](https://github.com/higan-emu/libco).

Basically it requires dropping down to the assembler-level for each supported
architecture to implement the context switching, as modifying the CPU context
registers and stack frame directly is not permitted by most sane programming
languages. But it's really not hard: an x86 context switch is only about twelve
lines of assembler code. 32-bit ARM is a mere three.

libco is ISC licensed and supports a very large amount of CPU architectures, so
you can skip having to do all of this work yourself if you want.

There are of course as many cooperative threading libraries as there are people
talking about cooperative threading, so feel free to use another, or make your
own.

For most languages there's probably an implementation of it somewhere. But
perhaps not always.

# Closing

In any case, I really believe cooperative threading gives my emulators a unique
advantage in having easier to read code by removing the burden of managing state
machines, and it makes it trivial to be clock-cycle accurate as well.

Again, I'm not saying this is the best way to write a retro emulator, but it's
served me very well.
