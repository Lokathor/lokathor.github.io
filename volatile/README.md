
# Volatile

So the other day on Twitter a person [posted a perfectly innocent
question](https://twitter.com/myrrlyn/status/1376586600637956102).

> does Rust's `Cell` type have volatile semantics. is `Cell` the appropriate
> type for describing memory that is modified by peripheral hardware.
>
> do i have to write a volatile cell crate. i do not wish to write `volcel` at
> my job but i will do it if i have to

To which I replied

> (1) Cell does not have volatile semantics (2) do not write a volcel crate
> there are several crate for a VolatileCell type already (3) do not use any
> VolatileCell crate they literally are all incorrect.

Now before we begin, full disclosure: I have my own [volatile handling
crate](https://docs.rs/voladdress), which could be seen as a "competing" crate
with the rest of the "VolatileCell" style crates. It offers an alternate style
of abstraction, which is slightly more annoying to use in some cases, but has
the main benefit that it's at least **not** incorrect.

Some may consider it slightly problematic that I have this conflict of interest.
I consider it problematic that even with said conflict, my tweet was not wrong.

The explanation is much longer than fits in a tweet, even than in fits in
*three* tweets, and twitter threads are hard to keep track of, so we get a
markdown file in a github pages instead.

## Step-by-step

To make sure everyone is on the same page, we have to cover a few different
things one at a time.

Everyone in the twitter thread that sparked this essay thing seemed to have
slightly different background knowledge levels, so let's go over the situation
one element at a time.

## `dereferenceable(<n>)`

First we have to talk about a [parameter
attribute](https://llvm.org/docs/LangRef.html#parameter-attributes) called
"dereferenceable". This is an attribute that can be on function parameters
within LLVM IR.
* Within the LLVM IR that rustc generates and send to LLVM, shared references
  (`&T`) and unique references (`&mut T`) *have this attribute*.
* At the same time, const pointers (`*const T`) and mutable pointers (`*mut T`)
  *do not have this attribute*.

What does the attribute do? Let's first look at the full quote:

> This indicates that the parameter or return pointer is dereferenceable. This
> attribute may only be applied to pointer typed parameters. A pointer that is
> dereferenceable can be loaded from speculatively without a risk of trapping.
> The number of bytes known to be dereferenceable must be provided in
> parentheses. It is legal for the number of bytes to be less than the size of
> the pointee type. The nonnull attribute does not imply dereferenceability
> (consider a pointer to one element past the end of an array), however
> dereferenceable(<n>) does imply nonnull in addrspace(0) (which is the default
> address space), except if the null_pointer_is_valid function attribute is
> present. n should be a positive number. The pointer should be well defined,
> otherwise it is undefined behavior. This means dereferenceable(<n>) implies
> noundef.

The key phrase is here:

> A pointer that is dereferenceable can be loaded from speculatively without a
> risk of trapping.

So when a function argument is dereferenceable LLVM is allowed to read it
without the CPU [trapping](https://en.wikipedia.org/wiki/Trap_(computing)).

The main thing that LLVM uses this attribute for is that it *can* choose to read
from the reference before you've literally written the read into your source
code. Imagine the following:

```rust
  fn check_and_read_ref_or_use_default(&self, r: &i32) -> i32 {
    if self.expensive_condition_check() {
      *r
    } else {
      i32::default()
    }
  }
```

The condition check takes time, the memory read will take some time, so LLVM
*might* decide to perform the read first, before the check has been computed.
Thanks to the magic of [Instruction-Level
Parallelism](https://en.wikipedia.org/wiki/Instruction-level_parallelism), the
checking operations can happen *while* the read operation is still pending. If
the check comes out to `true` at the end, you keep the memory read. If the check
turns up `false`, you just throw out that value and don't worry about it.
However, for this transformation to be a valid one, LLVM must know that it's
safe to issue the read operation without the CPU blowing up.

This is *just one* possible way that LLVM *might* take advantage of a function
argument being marked as dereferenceable. There are others, we won't go into
them all.

## Memory Mapped Input/Output (MMIO)

Memory Mapped IO is how the CPU interacts with the rest of the world. Basically
all that the CPU is good at is reading addresses, doing math, and writing
addresses. Instead of teaching the CPU to do more things, we just wire it up to
some special addresses for it to read and write.

Need to tell the SATA controller something? MMIO. Want to read a timer? MMIO.
Want to tell a gamepad to start or stop the rumble motors? MMIO.

* Astronaut 1: "Wait, it's all MMIO?"
* Astronaut 2: "Always has been."

When the CPU accesses an MMIO address, it's *not* writing out some value "to
memory" so that the value will be there later if and when it's needed.

Instead, MMIO accesses *can* have some sort of "side effect" to them.

What's a side effect? Well, that's deliberately vague. It depends on the device
that the CPU is communicating with. Most importantly though: MMIO reads and
writes **do not** have to match, both in terms of the raw data value and also in
terms of the side effect. In practice, the side effects of MMIO are usually
"fairly sane", but there's weird stuff if you look.

**Problem:** A dereferenceable value can be read earlier than intended. Since
*even a read* can have a side effect this means that you can get side effects
earlier than intended. Perhaps it's the case that you were supposed to take a
branch and never read the value at all, in which case you now have *more* side
effects than you were supposed to have.

**Solution:** Never, *ever* have references to MMIO.
* It doesn't matter what type the reference points to.
* It doesn't matter that you used `UnsafeCell`,
  [link](https://rust.godbolt.org/z/djx4b5Kzc).
* Never make a reference into MMIO, or the program is allowed to have extra read
  side effects. This can reset timers you didn't intend, clear flags you haven't
  actually looked at, or any number of other problems. Technically speaking this
  isn't Undefined Behavior in the normal "now your program can mean anything!"
  sort of way. It's just *wrong* behavior that's borderline impossible to debug.

This is why `&VolatileCell<T>` based abstraction crates are all incorrect. They
have a reference to the MMIO location, and so LLVM is allowed to potentially.

### Dereferenceable Gaiden: The Uninit Story

On a side note: some people asked "hey if LLVM can do speculative reads of
`&MaybeUninit<T>` isn't that a huge problem? What if it's not initialized yet!?"

The answer is that if you *unconditionally* read from that your
`&MaybeUninit<T>` then it's the same either way, and if you *conditionally* read
from the `&MaybeUninit<T>` and then LLVM discards the value in the path where
the condition didn't hold, well then it's fine. Because LLVM actually *can* do
things that Rust *can't* do (this is one of them).

As to your follow-up question, "what about atomics?", similar deal. LLVM has
rules that are slightly different from most langauges that compile using LLVM.
Inside of LLVM any data races return `undef` instead of causing UB. Actually
using an `undef` value can cause problems, but just *having* the `undef` in hand
and then throwing it out is fine. With atomics, any early read from
dereferenceable would actually *always* be discarded because the "actual" reads
you'd be doing as the programmer would be with atomic access instead of the
"standard" access that dereferenceable uses. So again it "just works out".

## Access Elision

Alright, so we're only ever using raw pointers for MMIO. Got it. Now it's all
fine from here on, right?

(pretend that there's a *Cool Bear Says* dialog in here)

> Hey what if we have this code:

```rust
unsafe fn foo(x: *mut i32) {
  *x = 7;
  *x = 8;
}
```

Uh, well, if we're storing a 7 to the pointed to location... and then also
storing an 8 to that same location, we can just store the 8... right? It'd be
fine to skip the step where we store the 7?

> Yeah, sure.
>
> What about this code here, What possible return values can we get from this
> function?

```rust
unsafe fn bar(z: *const bool) -> usize {
  let mut x = 0;
  while *z {
    x += 1;
    if x == 1000 {
      return x;
    }
  }
  x
}
```

If the pointer points to `true` the return value will always be 1000, and if the
pointer points to `false` it will always return 0. So we can just read once and
then skip the rest of the reads and the math and all that?

(end of *Cool Bear Says*)

These are examples of "access elision". A memory access is "elided" (removed)
from the program by the optimizer because the optimized program will behave the
same in the end as the unoptimized version did. (The time taken to compute the
intermediate steps isn't considered to be "a part of" the program, as long as
you get the correct answer.)

Access elision can happen with both references and with pointers.

> What the yotz Lokathor!? How the frell are we supposed to manage to get
> anything done if the compiler is adding and removing accesses all over the
> place all the time!

Yes, yes, I know.

## Are We Doomed?

I haven't said the word "volatile" in quite a long time by now, maybe that can
save us?

Let's read [the LLVM
docs](https://llvm.org/docs/LangRef.html#volatile-memory-accesses), maybe they
can help. That section is a few paragraphs long but it's really important that
you read and understand all of them.

> Certain memory accesses, such as `load`'s, `store`'s, and `llvm.memcpy`'s may
> be marked `volatile`. The optimizers must not change the number of volatile
> operations or change their order of execution relative to other volatile
> operations. The optimizers *may* change the order of volatile operations
> relative to non-volatile operations. This is not Java's "volatile" and has no
> cross-thread synchronization behavior.
>
> A volatile load or store may have additional target-specific semantics. Any
> volatile operation can have side effects, and any volatile operation can read
> and/or modify state which is not accessible via a regular load or store in
> this module. Volatile operations may use addresses which do not point to
> memory (like MMIO registers). This means the compiler may not use a volatile
> operation to prove a non-volatile access to that address has defined behavior.
>
> The allowed side-effects for volatile accesses are limited. If a non-volatile
> store to a given address would be legal, a volatile operation may modify the
> memory at that address. A volatile operation may not modify any other memory
> accessible by the module being compiled. A volatile operation may not call any
> code in the current module.
>
> The compiler may assume execution will continue after a volatile operation, so
> operations which modify memory or may have undefined behavior can be hoisted
> past a volatile operation.
>
> IR-level volatile loads and stores cannot safely be optimized into llvm.memcpy
> or llvm.memmove intrinsics even when those intrinsics are flagged volatile.
> Likewise, the backend should never split or merge target-legal volatile
> load/store instructions. Similarly, IR-level volatile loads and stores cannot
> change from integer to floating-point or vice versa.
>
> **Rationale:**
>
> Platforms may rely on volatile loads and stores of natively supported data
> width to be executed as single instruction. For example, in C this holds for
> an l-value of volatile primitive type with native hardware support, but not
> necessarily for aggregate types. The frontend upholds these expectations,
> which are intentionally unspecified in the IR. The rules above ensure that IR
> transformations do not violate the frontend's contract with the language.

## Line By Line Break Down

Let's go one sentence at a time.

> Certain memory accesses, such as `load`'s, `store`'s, and `llvm.memcpy`'s may
> be marked `volatile`.

Okay, so to LLVM it's *accesses* that are volatile. Not values, not types, not
particular parts of memory.

The edges of Rust are well known for "doing whatever LLVM does", and here is no
exception. To perform a volatile access we use the
[read_volatile](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html) or
[write_volatile](https://doc.rust-lang.org/std/ptr/fn.write_volatile.html)
functions. There's also methods with the same names directly on the pointer
types if you'd like to use those instead (const and mut pointers can read, only
mut pointers can write).

Not all languages are like this! In C (and I think C++?) there aren't special
volatile intrinsics. Instead the `volatile` keyword can be used as a modifier to
a pointer type to make all accesses of that pointer be volatile accesses.

> The optimizers must not change the number of volatile operations or change
> their order of execution relative to other volatile operations.

**This Is The Important Line!**

This is the line that makes volatile work exactly how we want.

* **LLVM cannot change the number of volatile operations.** If you put two
  volatile writes in a function, LLVM *must* have exactly two writes in the
  compiled version, with or without optimizations.
* **LLVM cannot change the order of volatile operations relative to other
  volatile operations.** So if you volatile write to location `x` and then
  volatile read from location `y`, the compiled *must* perform those two
  accesses in the order that you wrote down.

> The optimizers *may* change the order of volatile operations relative to
> non-volatile operations.

Okay, important detail, don't wanna forget this.

**Example:** Maybe we want to write a bunch of data into a buffer, then tell a
DMA unit to copy that buffer to another location. We'd need a
[compiler_fence](https://doc.rust-lang.org/core/sync/atomic/fn.compiler_fence.html)
to ensure that all the standard memory acceses that fill the buffer are done
before the volatile MMIO access which initiates the DMA. If we didn't put in a
`compiler_fence` the volatile access to initiate the DMA unit *could* be moved
before the buffer has all been filled in.

> This is not Java's "volatile"

Friends I am bordering on infinity years old (33), and if you think that I
remember one bit about the Java programming that I did in high school then I
have an NFT to sell you.

But I'm sure that some of you with more recent Java backgrounds might need to
have this distinction made for you.

> and has no cross-thread synchronization behavior.

This is more important than it might seem at first. Volatile accesses *can* data
race if your program has more than one core/thread going at once.

> A volatile load or store may have additional target-specific semantics.

Hey we've heard of those...

> Any volatile operation can have side effects, and any volatile operation can
> read and/or modify state which is not accessible via a regular load or store
> in this module.

This is still kinda hazy but I'd say the important part is that any volatile
operation (not just `store`) "can read and/or modify state". In other words, *a
read operation **can** change the world state*. It's good that LLVM allows this,
because that's certainly the case for some hardware even if LLVM doesn't allow
this, and it'd be a shame if we could simply never use LLVM with those devices.

> Volatile operations may use addresses which do not point to memory (like MMIO
> registers).

Perfect!

> This means the compiler may not use a volatile operation to prove a
> non-volatile access to that address has defined behavior.

This would probably be worth a post of its own, but basically LLVM tracks what
addresses are part of normal memory and what ones aren't during compilation. If
you use a volatile access it doesn't make that address count as part of the
"normal" memory.

> The allowed side-effects for volatile accesses are limited.

Fair.

> If a non-volatile store to a given address would be legal, a volatile
> operation may modify the memory at that address.

Okay, so why would you do a volatile access to a location that's not for MMIO?
Well, maybe you want to enforce that a block of memory is cleared to 0 before
you free it. Or maybe you want to set all your allocations to a debug memory
pattern so you can see during debugging when you've probably read into uninit
memory on accident (if all the bytes are `0xAB` or whatever). Sounds fringe, but
sure, it's not out of the question.

> A volatile operation may not modify any other memory accessible by the module
> being compiled.

Okay... okay... so we can't use a volatile access to change the memory in the
"normal memory" parts of the program.

To go back to a previous example: we can use volatile to tell a DMA unit to copy
*from* our buffer into another place outside the address space, but we can't use
volatile to copy *to* our buffer, because that would modify memory accessible
from the module being compiled.

(Note: If you **did** need to use MMIO to modify memory within the program's
standard address space you'd likely use inline assembly. It works more like a
function call to a black box, so MMIO within an inline assembly block has more
freedom in what it can do.)

> A volatile operation may not call any code in the current module.

Fair enough.

Like with this above, if we needed to do this we'd use an inline assembly block.

> The compiler may assume execution will continue after a volatile operation, so
> operations which modify memory or may have undefined behavior can be hoisted
> past a volatile operation.

We can't hide our UB behind volatile. Check.

> IR-level volatile loads and stores cannot safely be optimized into llvm.memcpy
> or llvm.memmove intrinsics even when those intrinsics are flagged volatile.

Oh, interesting, I guess we really should stabilize those other volatile ops.

> Likewise, the backend should never split or merge target-legal volatile
> load/store instructions. Similarly, IR-level volatile loads and stores cannot
> change from integer to floating-point or vice versa.

So if we tell it to store a `u32` all at once, we'll get a single `u32` write.
If we tell it to write two `u16` next to each other we'll get that instead.
Assuming, in both cases, that the target allows memory accesses of those sizes.

## Has This Ever *Actually* Broke Things Before?

Well, not to my knowledge.

I mean I'm just one guy but I kinda follow along with this subject as you may be
aware by now, and I have yet to see an example where LLVM actually did the wrong
thing it was allowed to potentially do.

So it's not the worst thing ever, but it is a fundamental incorrectness that
lurks in most embedded Rust code.

## Is There A Fix In Sight?

People are aware of the issue, there are several issues about it around github.

However, at this exact moment (2021-04-02-23:46 Mountain Time), there's not any
fix currently in the timeline.

Since we're in the lead up to the 2021 edition, it's unlikely to be fixed soon,
but it will be fixed eventually.

Like I said earlier, this has (as far as I know) never been demonstrated to
actually cause any problems in any real programs. It just *could* cause
problems, so it's on the list of things to fix eventually.
