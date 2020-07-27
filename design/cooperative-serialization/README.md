In my previous article on
[cooperative threading](/design/cooperative-threading), I went over
the benefits of emulating components as cooperative threads. The biggest
issue with this approach is serialization, or in emulation parlance, save
states.

In this article, I'll explore the issue in more detail, and propose working
solutions.

# Recap

What makes serialization difficult is that the state machine variables that
maintain where we are at within a function have been moved into the native stack
being used by each thread.

This stack data is extremely non-portable, and even if you were to save and load
the stack from disk, there's no guaranteed way to get a specific reserved memory
address across multiple runs of a program. And even if you could, it would
undermine the point of
[ASLR (address space layout randomization)](https://en.wikipedia.org/wiki/Address_space_layout_randomization).

There are however a few methods we can use, and in practice, a combination of
each method tends to work best in practice rather than just one.

# Example

Let's define a simple barebones cooperative threaded CPU core for this article:

```cpp
void CPU::main() {
    if(interruptPending) interrupt();
    instruction();
}

void CPU::instruction() {
    auto opcode = fetch();
    if(opcode == 0xa9) return opcodeLDAconst();
}

void CPU::opcodeLDAconst() {
    A = fetch();
    N = A & 0x80;
    Z = Z == 0;
}

void CPU::fetch() {
    step(2);
    auto data = bus.read(PC++);
    step(4);
    return data;
}

void CPU::step(uint clocks) {
    apu.clock -= clocks;
    while(apu.clock < 0) scheduler.switch(apu.thread);
}
```

In this example, the CPU core ends up four functions deep into the stack frame
before yielding to the APU core. Let us presume that the APU core runs for a
while, and then a save state needs to be captured.

Unfortunately, we're in the middle of executing a CPU instruction. If we were to
simply recreate the CPU thread, it would begin executing at the start of
`CPU::main()`, which is not what we want at all.

# Thread Alignment

One might thing to change the above code like so:

```cpp
void CPU::main() {
    scheduler.leave(Scheduler::Serialize);
    if(interruptPending) interrupt();
    instruction();
}
```

With the idea being that we keep running the emulation until every thread has
exited at the start of its main function.

The problem is this falls apart very quickly as the complexity of the emulator
grows. Even with only 2-3 threads, you'll quickly find that your emulator never
gets into a state where every thread is aligned perfectly, and waiting on this
to happen results in the the serialization function hanging forever waiting for
it.

# Method #1: Fast Synchronization

The first method is to simply stop allowing the scheduler to switch to other
threads while we are trying to synchronize (align) every thread for
serialization:

```cpp
void CPU::step(uint clocks) {
    apu.clock -= clocks;
    while(apu.clock < 0 && !scheduler.synchronizing()) scheduler.switch(apu.thread);
}
```

The above code breaks determinism by allowing the CPU to call `bus.read(PC)`
even though it might be ahead of the APU, and the APU may write to the address
that PC points at. In other words, our emulated components become
desynchronized.

On the surface, this doesn't seem like a huge loss: we are only running the CPU
code out of synchronization with the APU for, at most, a single CPU instruction.
But even that tiny amount can break some of the most sensitive games out there.

# Out-of-Order Execution

There's also a deeper issue that comes into play when you start using one of the
coolest use cases of cooperative threads: out-of-order execution.

In the above code, and just like a state machine, we are constantly switching
from the CPU to the APU every time any time passes at all.

But what if we know the CPU was reading from ROM that cannot change, or from a
memory range that the APU has no access to? In other words, there is no
possibility for the APU to change the byte read from the bus. We could then
change our example code like so:

```cpp
void CPU::fetch() {
    step(2);
    auto data = bus.read(PC++);
    step(4);
    return data;
}

void CPU::step(uint clocks) {
    apu.clock -= clocks;
}

uint8_t Bus::read(uint16_t address) {
    //this is ROM, it cannot be changed
    if(address < 0x8000) return rom.read(address);
    //this is internal RAM, the APU cannot access it
    if(address < 0xc000) return iram.read(address - 0x8000);
    //this is external RAM, the APU can modify it
    while(apu.clock < 0 && !scheduler.synchronizing()) scheduler.switch(apu.thread);
}
```

This can turn out to be a really big speedup, because the CPU is unlikely to be
executing instructions out the shared external APU RAM. But it also means that
the CPU may be hundreds, or even thousands, of instructions ahead in time from
the APU, simply because the CPU hasn't talked with the APU in a long time.

(In fact, the above example code isn't sufficient because when you do this, you
*must* eventually force-switch to the APU to prevent the APU from deadlocking
if the chips never communicate, but for the sake of example, I've left that out
here.)

So what happens when you allow the CPU to complete that one single read is that,
in rare cases, that read may happen thousands of instructions ahead of the APU,
which is a *far* more serious transgression of synchronization. Thus, method
1 (fast synchronization) will *definitely* start to break some games.

In any case, before moving on, let's look at what a fast serialization would
look like in code form:

```cpp
void System::run() {
    scheduler.setMode(Scheduler::Running);
    //resume whichever thread was last active at last scheduler.leave() call
    scheduler.switch(scheduler.active->thread);
}

void System::serializeFast() {
    scheduler.setMode(Scheduler::Synchronizing);
    scheduler.switch(cpu.thread);
    scheduler.switch(apu.thread);
    //all threads are now paused at the start of their main() functions.
    //we can safely serialize the system now and ignore their stacks.
    //when unserializing, we simply recreate new, empty stack frames.
}
```

# Method #2: Strict Synchronization

It's not a given that running a thread to its entry point will require advancing
another thread in order to not desynchronize, and we can detect if it does:

```cpp
uint8_t Bus::read(uint16_t address) {
    ...
    //this is external RAM, the APU can modify it
    while(apu.clock < 0) {
        if(scheduler.synchronizing() && apu.clock < 0) {
            //we need to advance the APU core. if the APU core were previously
            //aligned to its entry point, this may misalign it away from its
            //entry point again (in fact it probably will.)
            scheduler.setDesynchronized();
        }
        scheduler.switch(apu.thread);
    }
}
```

And now we can write a stronger serialization method that keeps trying until it
is able to reach the entry points of all threads. Although it's very unlikely
that every thread would reach this point if we constantly synchronized threads
after every clock tick from each thread, it becomes much more likely to occur if
we only synchronize when absolutely required.

```cpp
void System::serializeStrict() {
    scheduler.setMode(Scheduler::Synchronizing);
    while(true) {
        scheduler.switch(cpu.thread);
        if(scheduler.wasDesynchronized()) continue;
        scheduler.switch(apu.thread);
        if(scheduler.wasDesynchronized()) continue;
        //if we've reached this point, every thread is at its entry point.
        //furthermore, we have not broken determinism: hooray!
        break;
    }
}
```

However, the more threads we emulate, and the more those threads talk to each
other, the less likely this method will be to ever finish. You could set a
timeout on the loop of a few thousand tries or so before giving up and falling
back on the `serializeFast()` method.

# Determinism

Another issue with the strict method is that it takes potentially a lot more
time to complete. When you serialize the system, you ideally do not want to
advance time at all. Doing so anyway interferes with determinism.

Imagine you have a pre-recorded sequence of controller inputs to play back. In
other words, you're playing back a movie recording of a previously-played game.

The first time, you let the movie play normally, and the second time, you ask to
save a state halfway through. That small action may cause a very slight
desynchronization if we end up using the fast synchronization method, and that
could desynchronize the movie in a domino-like effect: we have broken the
determinism (repeatability) of the emulator just by saving a state.

Now imagine you want to implement something like real-time rewind, you would
need to be constantly serializing the system to have a history buffer to rewind
with. All of those potential desynchronizations start to add up.

# Method #3: Hibernation (Unsynchronized)

The final method is absolutely not portable, but if our cooperative threading
implementation allows us to allocate the stack ourselves, and the CPU register
context is stored somewhere in there, then we can simply serialize the entire
native stacks into our save states and restore them later ... with a few massive
caveats, of course.

This idea is basically a form of hiberation, in the way a computer might
physically power itself down, and then upon booting up again, restore actively
running programs right where you left off. But it's in a more limited context of
only applying to the portion of our program that was emulating a game. You
wouldn't, after all, want loading a save state to undo changed GUI settings like
the placement of the video output window, hotkey bindings, the volume output
setting, etc.

## Caveat #1: the location of the stack cannot move

You cannot delete and reallocate the stack memory when unserializing (loading a
save state), because it will likely end up at a different memory address. The
stack will contain relative pointers and addresses, and you've just invalidated
them.

This also means that you can't really save these hibernated states to disk for
later use, unless you have a way of always ensuring that each thread's stack is
allocated at the exact same memory address.

Now technically, this is *sort of* possible by using mmap with MAP_FIXED, but
it is all around a rather dangerous and unstable idea. And if your OS is using
ASLR, all of your program code location addresses will be invalidated anyway, so
this really isn't a very good solution for disk-based states.

## Caveat #2: threads cannot allocate dynamic memory

Let's say we changed our `CPU::main()` function above like so:

```cpp
void CPU::main() {
    static bool initialized = false;
    if(!initialized) {
        initialized = true;
        ram = (uint8_t*)malloc(8192);
    }
    scheduler.leave(Scheduler::Serialize);
    ...
}

void CPU::subfunction() {
    ram[0] = 1;
}
```

It's a pretty lousy example, but the stack may contain function call arguments
or even local variables that reference this dynamically allocated memory, and
when you unserialize a save state, you're going to end up with a different
address for `ram` above, and your stack will contain dangling pointers. Not
good.

The solution here is to allocate all memory required before starting the
emulator. If dynamic allocations are really necessary, a custom memory pool
allocator can be created, which cores can use to acquire and release heap memory
with as-needed, in a way that is deterministic across runs.

# Putting It All Together

If you use a combination of the above approaches, you can implement very
reliable and portable serialization.

When saving permanent states that should be written to disk, start with using
method 1 (fast serialization.) For games that have strong synchronization
requirements, fall back to method 2 (strict serialization.)

When saving temporary states that will never be stored to disk, but that require
determinism in order to work well, use method 3 (hibernation.) Good examples of
this would be for real-time rewind, run-ahead, and other such features.

Having to use the first two methods for disk-based saves is a bit less than
ideal, but you can imagine that an end-user will be capturing save states in
your emulator perhaps 1-10 times per minute at most. Usually they won't be
saving states actively at all.

Whereas real-time rewind and run-ahead require serializing the system every
single frame, or sixty times a second, or 3,600 times a minute, which is a
vastly bigger deal for determinism violations.

In practice with 12+ years of supporting save states in
[bsnes](https://github.com/bsnes-emu/bsnes) and [higan](https://github.com/higan-emu/higan), I have
never had a single report from a user where a manual save state (using methods 1
and 2) have resulted in failure. That doesn't mean it's impossible, but it does
signify that it's very rare.

It's only when capturing states every single frame, or 3,600 times per minute,
that exactly two games in the entire SNES library started to exhibit issues that
required method 2 (strict) synchronization. (Tales of Phantasia and Star Ocean,
if you were curious.)

With methods 1 and 2 combined, even rewind became rock-solid in my emulators,
but I still devised method 3 to ensure absolute determinism for the sake of
absolutely perfect run-ahead support.

Still, if deterministic states are truly essential and need to be unserializable
even across program runs, you *can* delve into using mmap with MAP_FIXED and
ASLR disabled to achieve this, but I really wouldn't recommend it.

# Closing

It's unfortunate that there's no silver-bullet to this problem that is 100%
portable, at least not without going into a full-blown custom virtual machine,
which would have tremendous performance implications.

But even with having to utilize these three methods for serialization, I still
find it to be personally worth the massive code clarity benefits of using
cooperative threads over state machines for emulator synchronization.

Ultimately though, the choice is yours which technique you choose to use.
