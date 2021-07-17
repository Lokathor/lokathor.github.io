# A Little Bit About PRNG Stuff

Let's talk about [Pseudorandom Number Generators][wp-prng], or PRNGs for short.

[wp-prng]: https://en.wikipedia.org/wiki/Pseudorandom_number_generator

Pseudorandom number generators are a deterministic way to take some **state** and perform a **step** so that you'll have a new state and get a random-seeming output value.

In Rust we'd generally put the state inside a struct and then have a method that produces an output value once per call.

```rust=
struct MyPrng {
    /* TODO: whatever fields */
}

impl MyPrng {
    fn next_output(&mut self) -> OutputType {
        todo!("implement MyPrng::next_output")
    }
}
```

## Evaluating A Generator

There's several categories that we might care about when considering a PRNG.

First of all we can consider the **size** of the generator's state. We want the size to be as small as possible, but when the size is *too small* then every other category will suffer.

Obviously we also want the **speed** of calculating each output to be as fast as possible.

When we talk about the **quality** of the output generator we mean how random-seeming each output is. Is the output biased, can the output be predicted based on the previous outputs, stuff like that. There's a lot of statistics work involved here and you're *not* expected to know it all yourself. Instead, there's test suites that are available that will run your generator and then do the statistics on the output and print out some results.

* There's [TestU01][github-TestU01], which is a C library that has the `SmallCrush`, `Crush`, and `BigCrush` test batteries.
* There's also [PractRand][sourceforge-PractRand], which can be used as a library or as a CLI tool.

[github-TestU01]: https://github.com/umontreal-simul/TestU01-2009/
[sourceforge-PractRand]: http://pracrand.sourceforge.net/

The **period** of the generator is how many times you can step the generator before the output sequence loops around. Since they're deterministic, eventually *all* PRNGs will loop, and things seem a lot less random if you've already seen the output sequence before. Even if you don't exhaust the full period of a generator, the apparent quality of the output tends to degrade once you've run through a significant portion of a generator's period. The period of a generator is *at most* equal to the number of possible states within the generator's overall state. If a generator's state is a single byte (8 bits) then it will have a period of *at most* `2**8`. However, not all generator state is always used with maximum efficiency towards having a long period, so sometimes the period is less than the maximum.

* Note: the notation `X**Y` is taken from Python, meaning "X to the power of Y". Since most programming languages use `^` to mean the XOR operation, and since this is meant to be programmer-oriented, we'll use `**` to mean exponent within this document.

A generator has **uniform** output if every possible output value appears an equal number of times in the full output sequence. It could be just once, or it could be more than once, as long as all outputs appear an equal number of times. This might seem very simple, but some generators don't do this.

Finally, a concept that's similar to uniformity, but not quite the same, is **k-dimensional equidistribution**. This means "how many times in a row can you call the generator before the next output can't be every possible value in the output sequence?". If you're using multiple generator calls aggregated together then you probably care about this one. Say you're generating 2D points, with one call for X and one for Y. I think you would probably want to know if the point `(1,1)` can't ever *actually* occur because the output sequence doesn't ever have an output of `1` twice in a row. Maybe that chance of actually seeing a `(1,1)` is *small*, but it should be *possible* right?

## My Kingdom For A Hardware Multiplier

The types of PRNGs we'll be focusing on today assume that your device is *at least* advanced enough to have hardware integer multiplication. This might seem like a low bar, but many fun retro devices don't have that, or they do but it's costly. If you're programming for something where multiplication is a heavy cost and you want something cheaper than that check out the [XorShift][wp-xorshift] family of generators instead.

[wp-xorshift]: https://en.wikipedia.org/wiki/Xorshift

Because of the details of how the XorShift generator family works they never actually output 0, which means that they all have non-uniform output.

## The Generators We Will Talk About Here Are Not Cryptographically Secure

When a pseudorandom number generator meets some high enough standards to make it suitable for cryptography we call it a [Cryptographically-secure pseudorandom number generator][wp-csprng]. This means that you can use the generator for keeping your secrets.

[wp-csprng]: https://en.wikipedia.org/wiki/Cryptographically-secure_pseudorandom_number_generator

The generators that we'll talk about here today **are not cryptographically secure**.

In other words, **do not use these generators for cryptography**.

I'm serious.

# Linear Congruential Generators

One of the easiest types of generator to understand and implement is the [Linear Congruential Generator][wp-lcg] (LCG). They're so simple to implement that they're used in countless programs.

[wp-lcg]: https://en.wikipedia.org/wiki/Linear_congruential_generator

The LCG family of generators was developed in 1958 by W. E. Thomson and A. Rotenberg, based on the slightly earlier Lehmer generator which had been developed by D. H. Lehmer in 1951. While there *are* some problems with this type of generator, they're well understood problems that we can compensate for.

## LCG Basics

The core formula of the LCG family is very simple. If you look at the wikipedia page they give it using the full fancy math notion, but the simple version is like this:

```
// pseudocode

next_state = (a * current_state + c) % m
```

So we'll multiply by an `a` parameter, and then add a `c` parameter, and then finally modulus divide that whole thing by an `m` parameter.

There are also some constraints on what we can pick for `a`, `c`, and `m` if we want it to work right:

* `m` > 0
* `a` > 0 && `a` < `m`
* `c` >= 0 && `c` < `m`
* `x` >= 0 && `x` < `m`

And this all assumes "actual math", where all numbers have infinite precision.

However, we want to use "computer math", where the numbers occupy bits and bytes somewhere, and modulus is a much more expensive operation than multiply or add. What we'll do is pick a particular integer type to do our math with and just let the hardware's natural wrapping operations give us the "modulus by `m`" step for free. For example, if we do `u32` operations then `m = 2**32`, and if we do `u64` operations then `m = 2**64`.

For `a` and `c`, we can use some const generics. The values are not intended to change during the life of any particular generator, so we'll bake them into the type to save on the amount of per-generator state we store. Instead of just `a` and `c`, we'll call the const generic values `MUL` and `ADD` in our code, since that's what they do, and it's much easier to understand with a real name on things. Mathematicians are the only people, on the Earth or off it, that are worse at naming things than programmers.

Speaking of names, our `u64` value that we're multiplying and adding to each step of the generator needs a name. I suppose we can call it the `position`. We can't just call it "state" because in a bit we'll see some handy add-ons for the LCG which also use their own state too. Every time you take a step, you're at a new position, so that makes enough sense, right? Maybe? Oh well, programmers *are* the 2nd worst people at naming things, after all.

```rust=
#[derive(Debug, Clone)]
#[repr(transparent)]
struct GenericLcg64<const MUL: u64, const ADD: u64> {
    position: u64,
}

impl<const MUL: u64, const ADD: u64> GenericLcg64<MUL, ADD> {
    fn next_u64(&mut self) -> u64 {
        self.position = self.position.wrapping_mul(MUL).wrapping_add(ADD);
        self.position
    }
}
```

## Selecting Our LCG Parameters

Our `position` value can be any `u64` at all, no prohibited values that make the generator stop working or anything like that. I always love having no prohibited values, keeps things simple.

The `MUL` and `ADD` parameters, on the other hand, if we read the Wikipedia page we see that they must be selected carefully.

If we want the maximum period possible for our generator, and given that we've already selected `m=2**64`, then we need to have `ADD != 0` and make sure that we satisfy the following rules (the "Hull–Dobell Theorem"):

* `m` and `ADD` are relatively prime (aka: `gcd(2**64, ADD) == 1`), which means `ADD` must be odd.
* `MUL-1` is divisible by all prime factors of `m`.
* `MUL-1` is divisible by 4, since `m` is divisible by 4.

Then we keep reading and find out

> Although the Hull–Dobell theorem provides maximum period, it is not sufficient to guarantee a *good* generator.

Gha!

Okay, well, there's a chart of common parameters...

> This table is to show popularity, not examples to emulate; *many of these parameters are poor.*

:/

> Tables of good parameters are available.\[2\]\[10\]

:D

Okay so let's check out those references numbered 2 and 10:

* 2: Steele, Guy; Vigna, Sebastiano (15 January 2020). "**Computationally easy, spectrally good multipliers for congruential pseudorandom number generators**". [arXiv:2001.05304](https://arxiv.org/abs/2001.05304) \[cs.DS\]. At this point it is unlikely that the now-traditional names will be corrected. Mathematics of Computation (to appear). Associated data at <https://github.com/vigna/CPRNG>.
* 10: L'Ecuyer, Pierre (1999). "**Tables of Linear Congruential Generators of Different Sizes and Good Lattice Structure**". Mathematics of Computation. 68 (225): 249–260. [CiteSeerX 10.1.1.34.1024](https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.34.1024). [doi:10.1090/S0025-5718-99-00996-5](https://www.ams.org/journals/mcom/1999-68-225/S0025-5718-99-00996-5/home.html). Be sure to read the [Errata](https://www.iro.umontreal.ca/~lecuyer/myftp/papers/latrules99Errata.pdf) as well.

Ah, hmm, well let's check the one from 2020 I suppose. Here's the [direct link to the PDF](https://arxiv.org/pdf/2001.05304.pdf) if you don't want to bother looking at the arXiv website and finding the right link on your own.

Okay, so, let's read that *Introducton* chapter... Here we go:

> In this paper, we provide lists of multipliers for both MCGs and LCGs, continuing the line
of work by L'Ecuyer in his classic paper.

Oh hey there's that github link... okay in the readme for the github it it says their database of numbers is 14GB compressed, and 36GB decompressed. Well, let's keep reading the paper?

*Chapter 2* talks about the Spectral Figures of Merit. Ohhhhh nooooo. Lattice, hyperplanes, spectral test, hermite constant, I'm very lost, but let's keep going.

*Chapter 3* is about Computationally Easy Multipliers. This is the interesting idea that if we're multiplying large-bit values by a constant, then an optimizing compiler can sometimes reduce the instructions needed to actually do the multiplication. This doesn't affect us right now since we're using 64-bit values on device that's assumed to also be 64-bit. However, if we moved up to a 128-bit LCG at some point we might want to keep this in mind. Or if we wanted to have a 64-bit LCG that was on a 32-bit device we might look back here too.

*Chapter 4*: Bounds on Spectral Scores, *Chapter 5*: Beyond Spectral Scores, these are chapters that again I don't really understand. I'm sure math people probably understand it. *Chapter 6*: Potency gives us another way to determine that a multiplier is not likely to be good. *Chapter 7* is about MCGs, so that doesn't really apply to us.

Ah, finally, *Chapter 8*: Tables. There's a wonderfully long explanation here of how all the different multipliers are ranked with spectral scores and harmonic scores and all sorts of things. The short version seems to be that, if we don't really know what we're doing:

* We can find the pages for the `m` value we're using.
* Make sure we're using the page that says `M+` at the top of the 3rd column (not `M*`, that's for MCG and we're doing LCG).
* Then pick a multiplier bit width, which can be less than your full LCG's bit width. As they mentioned in Chapter 3 of the paper, there might be an advantage to picking a smaller bit width multiplier. We'll investigate that in a bit.
* Finally, pick the 2nd value in each grouping of four numbers for that bit width.

So for an LCG with `u64`, we look in the chart and find `0xAF251AF3B0F025B5`. There's other values in the list we could also use, but we'll use that one for now. We finally have a `MUL` value.

For our `ADD` value it's much simpler: we can just pick 1. Really. We can pick any odd number, so for the moment it's fine to just use 1. Why pick 1 over any other odd value? Well, it's the same train of thought as that "Computationally Easy Multipliers" stuff. The details vary by CPU, but usually an instruction set can encode a small "immediate" value directly into an `add` instruction, without loading a value into a separate register. We're picking 1 so that *most of the time* we can skim another small boost out of it. If it doesn't hurt us, we might as well pick up the micro-optimziation.

This lets us fill in some standard values for `MUL` and `ADD`.

```rust=
type StandardLcg64 = GenericLcg64<0xAF251AF3B0F025B5, 1>;
```

Let's check on our generator's stats:

* Speed: blazing fast
* Size: 64-bit
* Period: `2**64`
* Output Quality: Bad
* Uniform? Yes
* Equidistribution: 1

Hm. Not so great.

## Reducing The Output Size

The problem that we've got is that, with `m=2**64`, every bit in the generator's state is on a `2**bit` cycle, counting from 1 as the lowest bit. So the highest bit is on a `2**64` cycle, the next bit is on a `2**63` cycle, and so on down to the lowest bit being on a `2**1` cycle. It's just constantly flipping between 0 and 1. That means that quite a few of our bits are *extremely* predictable. There an exponential decay in randomness quality *per bit*.

But, you know, in practice we usually don't need more than a `u32` of ourput per individual use of a PRNG. That's "big enough" for plenty of practical purposes. A random boolean just needs one bit, a random index into an array usually needs far less potential options than `u32::MAX`, etc etc.

So what we'll do is have our "LCG64" only output the top 32 bits of each step of the generator. This is called an "LCG64/32".

```rust=
#[derive(Debug, Clone)]
#[repr(transparent)]
struct GenericLcg64_32<const MUL: u64, const ADD: u64> {
    position: u64,
}

impl<const MUL: u64, const ADD: u64> GenericLcg64_32<MUL, ADD> {
    fn next_u32(&mut self) -> u32 {
        let out = (self.position >> 32) as u32;
        self.position = self.position.wrapping_mul(MUL).wrapping_add(ADD);
        out
    }
}

type StandardLcg64_32 = GenericLcg64_32<0xAF251AF3B0F025B5, 1>;
```

Here, we're taking the top 32 bits of the position as an output value and then computing the next position. As another micro-optimization, we're using the last steps's position value to make the output value instead of the current step. On any device with [instruction level parallelism][wp-ilp], which is most of them these days, this lets the shift happen concurrently with the multiply instead of depending upon the multiply. Again, it's a very small difference, but we might as well.

[wp-ilp]: https://en.wikipedia.org/wiki/Instruction-level_parallelism

The generator is effectively just as fast as before, it's just as big with the same period. The outputs are still uniform, and we're still only 1-dimensional. However, our quality *does* jump up a little bit. At this point we could pass the `SmallCrush` tests from `TestU01`.

<!--

### Testing With SmallCrush

Okay, so we expect that our generator should be not the worst thing ever. It won't be able to pass a full test suite yet, but it should at least be able to do the SmallCrush tests. Let's verify that before we continue.

Thankfully, there's already a blog post on [how to test with TestU01][how-to-test-with-TestU01], so we'll mostly go by that.

One problem, the TestU01 setup assumes that we're testing C code. Heck, I don't even want to think about how we'd make it usable from Rust, we'll just write the same formula in C:

```c=
// adapted from https://www.pcg-random.org/posts/how-to-test-with-testu01.html

#include "TestU01.h"
#include <stdint.h>

static uint64_t y = 2456;

uint32_t lcg64() {
  uint32_t out = (uint32_t)(y>>32);
  y = y * 0xAF251AF3B0F025B5 + 1;
  return out;
}

int main() {
  unif01_Gen *gen = unif01_CreateExternGenBits("lcg64 blog", lcg64);
  bbattery_SmallCrush(gen);
  return 0;
}
```

Okay, now we build the program from within our `/TestU01` directory.

```shell=
gcc -std=c99 -Wall -O3 -o lcg64 lcg64.c -Iinclude -Llib -ltestu01 -lprobdist -lmylib -lm
```

And finally we run the thing.

A whole ton of stuff spits out, and then at the end we see

```
========= Summary results of SmallCrush =========

 Version:          TestU01 1.2.3
 Generator:        lcg64 blog
 Number of statistics:  15
 Total CPU time:   00:00:07.65

 All tests were passed
```

Yatta!

An LCG64/32 can't pass the BigCrush test suite, no matter the multiplier you pick, so we'll stop there for now. Always good to do a spot-check though.

### Jumping The Generator

Here's a fun trick: what if we wanted to go farther ahead in the sequence? Not like a little ahead, like a lot ahead. Like 500,000 steps ahead, or 1,000,000 steps ahead. We don't want to *actually* run the generator that many times and use all that CPU just to get into position and start the real simulation.

Well it turns out that M. E. O'Neill over at <https://www.pcg-random.org/> has the LCG64/32 on their generator summary chart, and listed is the ability for it to jump ahead. Interesting. In the download page there's a link to a [github](https://github.com/imneme/pcg-c) for the C implementation of the related PCG stuff, which can also jump ahead. If we look in the `src/` directory there's all these [pcg-advance](https://github.com/imneme/pcg-c/blob/master/src/pcg-advance-64.c) files. In that file it says:

```c=
/*
 * The method used here is based on Brown, "Random Number Generation
 * with Arbitrary Stride,", Transactions of the American Nuclear
 * Society (Nov. 1994).
 */
```

Neato. If we look up that paper name we get ourselves to the [OSTI page for the paper][osti-lcg-jump], which... I guess we click [find in google scholar][google-scholar-brown-94]? Right and... okay here's a [PDF Link](https://mcnp.lanl.gov/pdf_files/anl-rn-arb-stride.pdf), finally.

Okay so we're talking LCGs still, but the variable names are a little different this time:

```
s_(x+1) = (s_(i) * g + c) `mod` 2**m
```

So now we've got `g` as the name of our multiplier, and `c` is still the name of our increment value. Also, within the paper `m` is the power of 2, not the full value, so `m` would be `64`, not `2**64`.

Next we've got a formula to jump a generator `k` steps:

```
s_(k) = (s_(0) * g**k + c * (g**k - 1)/(g-1)) `mod` 2**m
```

However, there's a problem with this formula: for it to work we need to compute the intermediate steps without loss of precision (mostly those `g**k` parts). In other words, we can't have any value wrapping on any of our intermediate steps. This requires that the whole thing be done with at least double the bit width of the LCG state. A less than ideal requirement. We *could* use it with an LCG64/32 by using `u128` math, but if we wanted to move up to an LCG128/32 or something else later on we'd be stuck. Rust doesn't offer a `u256` type. Even if it did offer `u256`, emulating 256-bit math using 64-bit ints isn't gonna be a great time.

Oh, thank Ferris, if we turn to the next page it says there's some alternate formulas we can use:

```
G = g**k `mod` 2**m
  = some sort of summation `mod` 2**m
  = some sort of product `mod` 2**m
  // I don't know how to write those out well,
  // if you wanna know the details look in the paper

C = c * (g**k - 1)/(g-1)) `mod` 2**m
  = c * (1 + g + g**2 + ... + g**(k-1)) `mod` 2**m
```

And we also get some formulas for how to compute that stuff on a computer! Radical! Here's some pseudocode of what it says in the paper:

```
Algorithm G:
  G <- 1
  h <- g
  i <- (k + 2**m) `mod` 2**m
  while i > 0:
    if i is_odd:
      G <- (G * h) `mod` 2**m
    h <- (h * h) `mod` 2**m
    i <- i `intDiv` 2
  return G

Algorithm C:
  C <- 0
  f <- c
  h <- g
  i <- (k + 2**m) `mod` 2**m
  while i > 0:
    if i is_odd:
      C <- (C * h + f) `mod` 2**m
    f <- (f * (h+1)) `mod` 2**m
    h <- (h * h) `mod` 2**m
    i <- i `intDiv` 2
  return C
```

And for those of you at home paying close attention to those loop conditions, you may have noticed that we can combine quite a bit of the initaliazation code and the loop as well. Let's try this in Rust.

```rust
pub const fn lcg_u64_jump(state: u64, base_mul: u64, base_inc: u64, jump_count: u64) -> u64 {
    let mut big_g = 1_u64;
    let mut h = base_mul;
    let mut big_c = 0_u64;
    let mut f = base_inc;
    let mut i = jump_count;
    while i > 0 {
        if (i & 1) != 0 {
            big_g = big_g.wrapping_mul(h);
            big_c = big_c.wrapping_mul(h).wrapping_add(f);
        }
        f = f.wrapping_mul(h.wrapping_add(1));
        h = h.wrapping_mul(h);
        i /= 2;
    }
    state.wrapping_mul(big_g).wrapping_add(big_c)
}

/// Matches with the sequence from `lcg_u64_32_easy_step`
pub const fn lcg_u64_easy_jump(state: u64, jump_count: u64) -> u64 {
    lcg_u64_jump(state, 0xAF251AF3B0F025B5, 1, jump_count)
}
```

We can also use this to go "backward" too, by simply going extra far forward, since the sequence is a loop. If we want to go back by 1 step, we'd just do `-1_i64 as u64`, or similar for whatever other jump value we want.

This is all kinda tricky, so let's be sure to [run a bit of testing](https://play.rust-lang.org/?version=stable&mode=release&edition=2018&gist=7623c2b8d9e8b972f6d55bf363d98586) over this code:

```rust=
#[test]
fn test_lcg_u64_jump() {
    let s = 5678;
    let (next_state, _) = lcg_u64_32_easy_step(s);
    let (next_state2, _) = lcg_u64_32_easy_step(next_state);
    let (next_state3, _) = lcg_u64_32_easy_step(next_state2);

    let s_jumped = lcg_u64_easy_jump(s, 1);
    assert_eq!(s_jumped, next_state);

    let next_back_one = lcg_u64_easy_jump(next_state, -1_i64 as u64);
    assert_eq!(next_back_one, s);

    let s_jumped2 = lcg_u64_easy_jump(s, 2);
    assert_eq!(s_jumped2, next_state2);

    let s_jumped3 = lcg_u64_easy_jump(s, 3);
    assert_eq!(s_jumped3, next_state3);
}
```

### Assembly Check

Let's also have a glance at the assembly we get just to see if things seem reasonable.

[x86_64-pc-windows-msvc](https://rust.godbolt.org/z/rE5Kce36W)

```asm
example::lcg_u64_32_easy_step:
        movabs  rax, -5826220908985965131
        imul    rax, rcx
        add     rax, 1
        shr     rcx, 32
        mov     edx, ecx
        ret
```

Not bad at all, basically what we expect. We do one multiply, one add, and one shift. The add is even using an immeidate value like we'd hoped it would.

[aarch64-unknown-linux-gnu](https://rust.godbolt.org/z/9eYsMvbG4)

```asm
example::lcg_u64_32_easy_step:
        mov     x8, #9653
        movk    x8, #45296, lsl #16
        movk    x8, #6899, lsl #32
        movk    x8, #44837, lsl #48
        orr     x9, xzr, #0x1
        madd    x8, x0, x8, x9
        lsr     x1, x0, #32
        mov     x0, x8
        ret
```

Oooohhh, oof. The `a64` code is having a rough time with that huge multiplier. If we switch to a smaller suggested multiplier from the table (`0xf691b575`) then we can reduce the setup by two instructions like they said:

```asm
@ alternate multiplier: 0xf691b575
example::lcg_u64_32_easy_step:
        mov     w8, #46453
        movk    w8, #63121, lsl #16
        orr     x9, xzr, #0x1
        madd    x8, x0, x8, x9
        lsr     x1, x0, #32
        mov     x0, x8
        ret
```

I guess... I sorta expected it to just put in a `u64` constant at the end of the function and then load that constant with an offset load using the program counter. I guess that doesn't really inline as well, maybe? I dunno, I'm not a compiler person.

This certainly makes me inclined to switch to a smaller multiplier. I'd like our generator to work as well as possible on ARM. However, our generator isn't yet high enough quality to pass any of the test suites. Until we're passing one of the random test suites, I'm going to hold off on the multiplier change. We'll get back to this for sure.

### Multiple Streams

One thing we might care to do is use *multiple* output streams, with a separate stream for each generator. This doesn't increase the output period per generator, but it does prevent us from over using any particular part of the generator's output sequence. The sequence will end up seeming less random the more you see it over and over, so it's generally good to spread out what parts of the sequence we're using.

To have multiple output streams is very simple: We just change the `c` value, the amount we're adding per step. We just have to make sure that it remains odd. There's two basic ways to do this:

* `(c | 1)` will ignore the lowest bit by forcing it on.
* `((c << 1) | 1)` will ignore the highest bit by discarding it in the shift, and then we force the lowest bit on.

Both of these will force the low bit on, but the second expression will work better with common user inputs. What I mean is that people will often pick small digit values when they have to pick a number by hand. This means that the high bit is almost never important, but the lowest bit is always important. By picking the second expression we let the user see an output difference when they change the stream value from `2` to `3` or other single step changes like that. Technically this requires one additional operation, but that's only a single cycle. It's also a loop-invariant value, so when using the generator rapidly to generate large quantities of random numbers we don't pay the extra cost each pass of the loop.

### Extending The Period

Here's a nice trick from the [PCG paper][pcg-paper], chapter 7. It's intended for use with PCGs, which we'll get to in a moment. However, the technique also works with LCGs, so this one technique we'll discuss right now.

The basic idea is that we're going to add some more state to our generator, and then use that state to mask the output.

> As usual, let's suppose we have a `b`-bit generator producing `r`-bit values, which we shall call the *base generator*. The *extended generator* consists of those `b`-bits of state and `k` additional `r`-bit values (where `k` >= 1), which we shall call the *extension array*.

So for us, we're a `b=64` bit generator producing `r=32` bit values, and then we have a `k` element array full of `r`-bit values. So our state changes from being just a `u64` to being a struct with two fields. The `u64` we'll call the "position", and then we have that array too:

```rust=
/// Note: K should always be a power of 2!
pub struct Lcg64Ext<const K: usize> {
    position: u64,
    extension: [u32; K]
}
```

> The final output of the combined generator is produced by xoring the selected value from the extension array with the random number produced by the base generator.

Okay so every time we get a `u32` to output, instead of passing that out directly, we take a few bits from the current position value and use that to index into our extension array, and then we `xor` our intermediate output value with the value in the extension array to get the final output. That's why there's the note that `K` should be a power of 2, then we can just mask out a few bits and instantly have a valid index without having to do a modulus operation.

We could use the highest set of bits, which makes the extension array usage very chaotic, or we could use the lowest set of bits, which would make the extension array usage very predictable. I'm going to go ahead and use the lowest bits for simplicity sake.

There's one very important final detail: we have to update the array in some way. Just having the array and doing the xor step without ever changing the array makes an output sequence that's only the normal length (though in some new ordering). If we update the array in a regular way then the cycle of the array combines with the cycle of the generator's position to give us a very large state overall. Instaed of a period of `2**b`, we'll have a period of `2**(k*r+b)`. If we add an extension array of four `u32` elements, then our period would go from `2**64` to `2**192`. That's *pretty big*, and that's just four elements. We could go even bigger than that if we wanted to.

The main rule for how to update the extension array is that you must update it in some regular way that will eventually cycle around, and you must update it *at least once* per cycle of the generator's position. The simplest system for updating the array is to just add 1 to the lowest element and let any overflow carry up from one array element to the next, with the highest element simply wrapping. The simplest way to update the array once per generator cycle is to just update the array when the generator's position is 0.

A step function for our extended generator would look something like this:

```rust=
// Note: K should be a power of 2!
pub const fn lcg_u64_32_ext<const K: usize>(s: Lcg64Ext<K>) -> (Lcg64Ext<K>, u32) {
    let next_position = s.position.wrapping_mul(0xAF251AF3B0F025B5).wrapping_add(1);
    let temp_out = (x >> 32) as u32;
    let ext_index = (s.position as usize) & K;
    let out = temp_out ^ s.extension[ext_index];
    let next_state = Lcg64Ext {
        position: next_position,
        // branch predictor, hear my prayers
        extension: if s.position == 0 {
            let mut a = s.extension;
            let mut carry = true;
            for a_mut in a.iter_mut() {
                if carry {
                    let (a_mut_next, carry_next) = a_mut.overflowing_add(1);
                    *a_mut = a_mut_next;
                    carry = carry_next;
                } else {
                    break;
                }
            }
            let mut i = 0;
            while i < a.len() && carry {}
            a
        } else {
            s.extension
        },
    };
    (next_state, out)
}
```

There is one downside to the extension array technique: it doesn't cleanly interact with the jumping ability we had earlier. When we do a jump it will only affect the `position` value, with no simple way to tell if our `extension` array should change or not too. I'm not sure it's a big deal. It might be good enough to just use either jumping or period extension but not both in the same generator.

Or we could even use both techniques in the same generator and just accept a little bit imperfection in the system.

Alternately, instead of using the `position` jump function we got from the Brown paper, we could just directly perform our extension array update. This would jump us forward in the sequence by however many steps normally happen per extension array update. In the case here where it's one update per `2**64` steps, then each update is `2**64` steps forward. Because we chose an easy to repeat update function (+1) we can also jump more than one block at a time. Since our update function is revesable (-1) we can easily jump backward as well.

# Permuted Congruential Generators

Okay we've got our `LCG64/32`, and we have several bonus techniques we can do with it as well. However, it's not so great on the output quality. How can we improve the generator quality further? I'm so glad that you asked, because M. E. O'Neill and that [PCG paper][pcg-paper] from before have the answer.

It's fairly simple: we're already running a minor "improving step" on the LCG's 64-bit output (`temp>>32`), and it's significantly improving the quality of our final 32-bit output. We just need to work on that improving step, and we can keep the rest of the generator stuff we've got so far.

### Testing with BigCrush

TODO

### Testing With PractRand

TODO

### Change To A Smaller Multiplier

TODO: smaller multipliers work better on ARM.

## Generating Bounded Integers

TODO

## Generating Normalized Floats

TODO

## Seeding Your Generator

TODO

[wp-pcg]: https://en.wikipedia.org/wiki/Permuted_congruential_generator
[pit-of-success]: https://blog.codinghorror.com/falling-into-the-pit-of-success/
[how-to-test-with-TestU01]: https://www.pcg-random.org/posts/how-to-test-with-testu01.html
[osti-lcg-jump]: https://www.osti.gov/biblio/89100-random-number-generation-arbitrary-strides
[google-scholar-brown-94]: https://scholar.google.com/scholar_lookup?title=Random+number+generation+with+arbitrary+strides&publication_year=Sat+&author=F.+Brown&journal=Transactions+of+the+American+Nuclear+Society
[pcg-paper]: https://www.pcg-random.org/pdf/hmc-cs-2014-0905.pdf

-->