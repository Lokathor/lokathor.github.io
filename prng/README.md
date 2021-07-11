# A Little Bit About PRNG Stuff

Let's talk about [Pseudorandom Number Generators][wp-prng], or PRNGs for short.

Pseudorandom number generators are a deterministic way to take some state and perform a step so that you get a new state and also an output value.

In Rust we might have a general outline like this:
```rust=
type State = (); // TODO: pick a state type
type Output = (); // TODO: pick an output type

pub const fn example_generator(state: State) -> (State, Output) {
    // TODO: implement example_generator
    ((), ())
}
```

Usually we would make a struct to hold the generator state, and make the stepping function be a method on that struct, and so on and so forth. Today we'll just have a lot of functions, that's fine.

### Rating A Generator

PRNGs are functions, and so just like with all our functions we want them to run as fast as possible while requiring as little data as possible, just like with any other function. These are the **speed** and **size** of the generator.

However, our `example_generator` function requires no actual input data and returns instantly. Since we didn't even do anything yet, there's got to be more criteria that a good generator has to meet.

In addition to being graded on speed and size, a PRNG should perform well in the following categories:

* The **quality** of the generator measures how well the generator's output can pass various statistical tests for randomness. The more tests that are passed the better the generator. You're not expected to know how to do any of these the statistical tests yourself. Instead, people have set up test suites that you can run any generator through. The two standard test suites for PRNGs are [TestU01][github-TestU01] (which contains the "SmallCrush" and "BigCrush" test suites) and [PractRand][sourceforge-PractRand]. They both just use standard input to read generator values over and over and eventually print out your results. The quality of a generator is usually some fuzzy words like "bad" or "good", or you can get more specific words like "passes BigCrush".
* The **period** of the generator measures how long until the generator's output sequence loops. Remember that these are *deterministic* functions, so each input state produces exactly one next state and one output value. When you feed that next state back into the function enough times, you'll eventually get to the state that you started with, which means your output sequence will have looped around as well. The maximum possible period of any generator is the generator's state size in bits. Example: If there's 8 bits of state that's `2**8` possible generator states, and so your generator can have at most a period of 256. However, most generators don't use all their state bits perfectly efficiently towards having a maximum period, so the actual period of a given generator is usually some amount less than the full number of bits in the generator state.
* The **equidistribution** of the generator measures how many times you can use the generator in a row before some part of the output range *can't* occur in the next output, no matter where you are in the output sequence. This is very important if we're using more than one step in the generator to make an aggregate value. Imagine we're generating random 2D points with two generator calls: if the point `(1,1)` can't ever actually occur because the output sequence never produces the value `1` twice in a row, we'd really want to know that ahead of time. This is also known as "k-dimensional equidistribution" for whatver value of "k".

Oh, and the notation `X**Y` is taken from Python, meaning "X to the power of Y". Since most programming languages use `^` to mean the XOR operation, we'll use `**` to mean exponent.

### These Generators Are Not Cryptographically Secure

When a pseudorandom number generator has high enough output quality that it can safely be used with cryptography, we call it a [Cryptographically-secure pseudorandom number generator][wp-csprng]. This means that you can use the generator for keeping your secrets.

The generators that we'll talk about here today **are not cryptographically secure**.

In other words, **do not use these generators for cryptography**.

### My Kingdom For A Hardware Multiplier

The types of PRNGs we'll be focusing on today assume that your device is *at least* advanced enough to have hardware multiplication. This might seem like a low bar, but many fun retro devices don't have a hardware multiplier. If you're programming for something very old that doesn't do multiplication, you probably want to look into the [XorShift][wp-xorshift] family of generators instead.

Because of the details of how the XorShift generator family works they never actually output 0, which means that they all have an equidistribution rating of 0. To me, that immediately puts them into the "only use it if you have to" category, so we won't spend any more time on them here.

## Linear Congruential Generators

One of the easiest types of generator to understand and implement is the [Linear Congruential Generator][wp-lcg] (LCG). It was developed in 1958 by W. E. Thomson and A. Rotenberg, based on the Lehmer generator which had been made by D. H. Lehmer in 1951.

Because they're so simple to implement they're used in countless programs. Even [pokemon][bulbapedia-rng] uses an LCG for random numbers (starting in Gen 3).

### Basics

The basic formula is easy enough, but this markdown document doesn't do fancy math rendering, so try to make sense of this:
```
x_(n+1) = ( a * x_(n) + c ) `mod` m
```

We have our current state, `x_(n)` (`x` with a subscript `n`), and to get our next state `x_(n+1)`, we just multiply by `a`, add `c`, and then modulus the whole thing by `m`. Of course this assumes that we're using "real math" without hardware bit width limits.

If we had some sort of "BigInt" type that could do infinite precision integer math, a function to step our generator once would look like this in Rust:
```rust
use assume_some_math_crate::BigInt;

/// Steps an LCG using BigInt values.
///
/// * `m` > 0
/// * `a` > 0 && `a` < `m`
/// * `c` >= 0 && `c` < `m`
/// * `x` >= 0 && `x` < `m`
pub fn lcg_bigint_step(x: BigInt, a: BigInt, c: BigInt, m: BigInt) -> (BigInt, BigInt) {
    let next_state = ((x * a) + c) % m;
    let out = next_state;
    (next_state, out)
}
```

Okay, we kinda got this. I always find it much easier to understand math once it's been converted away from normal math notation (gross!) and into nice and friendly programming code (yeah!).

This is the *most general* version of the concept. In fact it's so general that we have to put notes on the function about what the correct input values are. Let's try to fix this up so it's a little harder to accidentally misuse, [pit of success][pit-of-success] style.

We just need a small change: get rid of `m`. See how it's appearing in the rules of what every other value should be like? And then in the actual code, it's causing a modulus division, which is gonna be really slow.

To cut out the `m` value we just need to change our number type. If we use one of the standard unsigned integer types then:
* `m` will be `Type::MAX` + 1 (which can't actually be reprisented within the type).
* The `a`, `c`, and `x` values will automatically be less than `m`.
* `c` and `x` will automatically be `>=0`, we just have to caution people against having `a` be 0.
* The multiply and add steps will automatically be modulus `m` in the hardware, at no extra cost.

Since most systems these days are 64-bit that's the biggest size integer that we can multiply in hardware, so we'll use `u64` values for this next version.

```rust=
/// Steps an LCG using u64 values.
///
/// * `a` > 0
pub const fn lcg_u64_step(x: u64, a: u64, c: u64) -> (u64, u64) {
    let next_state = x.wrapping_mul(a).wrapping_add(c);
    let out = next_state;
    (next_state, out)
}
```

Much better.

### Selecting Our Parameters

So our state value is `x`, and that will change from call to call. It can be any `u64` we want.

If we read the wikipedia page for LCGs though, we see that the `a` and `c` values are a little more sensitive.

If we want the maximum period possible for our generator, and given that we've already selected `m=2**64`, then we need to have `c != 0` and make sure that we satisfy the following rules (the "Hull–Dobell Theorem"):

* `m` and `c` are relatively prime (aka: `gcd(m, c) == 1`), which means `c` must be odd.
* `a-1` is divisible by all prime factors of `m`.
* `a-1` is divisible by 4, since `m` is divisible by 4.

Then we keep reading and find out

> Although the Hull–Dobell theorem provides maximum period, it is not sufficient to guarantee a *good* generator.

Gha!

Okay, well, there's a chart of common parameters

> This table is to show popularity, not examples to emulate; *many of these parameters are poor.*

:/

> Tables of good parameters are available.\[2\]\[10\]

:D

Okay so let's check out those references numbered 2 and 10:

* 2: Steele, Guy; Vigna, Sebastiano (15 January 2020). "Computationally easy, spectrally good multipliers for congruential pseudorandom number generators". [arXiv:2001.05304](https://arxiv.org/abs/2001.05304) \[cs.DS\]. At this point it is unlikely that the now-traditional names will be corrected. Mathematics of Computation (to appear). Associated data at <https://github.com/vigna/CPRNG>.
* 10: L'Ecuyer, Pierre (1999). "Tables of Linear Congruential Generators of Different Sizes and Good Lattice Structure". Mathematics of Computation. 68 (225): 249–260. [CiteSeerX 10.1.1.34.1024](https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.34.1024). [doi:10.1090/S0025-5718-99-00996-5](https://www.ams.org/journals/mcom/1999-68-225/S0025-5718-99-00996-5/home.html). Be sure to read the [Errata](https://www.iro.umontreal.ca/~lecuyer/myftp/papers/latrules99Errata.pdf) as well.

Ah, hmm, well let's check the one from 2020 I suppose. Here's the [direct link to the PDF](https://arxiv.org/pdf/2001.05304.pdf) if you don't want to bother looking at the arXiv website.

Okay, so, let's read that Introducton chapter... Here we go:

> In this paper, we provide lists of multipliers for both MCGs and LCGs, continuing the line
of work by L'Ecuyer in his classic paper.

Oh hey there's that github link... okay in the readme for the github it it says... their database of numbers is 14GB compressed, and 36GB decompressed. Well let's keep reading the paper.

Chapter 2 talks about the Spectral Figures of Merit. Ohhhhh nooooo. Lattice, hyperplanes, spectral test, hermite constant, I'm very lost, but let's keep going.

Chapter 3 is about Computationally Easy Multipliers. This is the interesting idea that if we're multiplying large-bit values by a constant, then an optimizing compiler can sometimes reduce the instructions needed to actually do the multiplication. This doesn't affect us right now since we're using 64-bit values on device that's assumed to also be 64-bit. However, if we moved up to a 128-bit LCG at some point we might want to keep this in mind. Or if we wanted to have a 64-bit LCG that was on a 32-bit device we might look back here too.

Chapter 4: Bounds on Spectral Scores, Chapter 5: Beyond Spectral Scores, these are chapters that again I don't really understand. I'm sure math people probably understand it. Chapter 6: Potency gives us another way to determine that a multiplier is not likely to be good. Chapter 7 is about MCGs, so that doesn't really apply to us.

Ah, finally, Chapter 8: Tables. There's a wonderfully long explanation here of how all the different multipliers are ranked with spectral scores and harmonic scores and all sorts of things. The short version seems to be that, if we don't really know what we're doing:

* We can find the pages for the `m` value we're using,
* Make sure we're using the page that says `M+` at the top of the 3rd column (not `M*`),
* Then pick a multiplier bit width (which can be less than your full LCG width),
* Finally, pick the 2nd value in each grouping of four numbers for that bit width.

So for an LCG with `u64`, we look in the chart and find `0xAF251AF3B0F025B5`. There's other values we could also use, but we'll use that one for now. We finally have an `a` value.

For our `c` value it's much simpler: we can just pick 1. Really. We can pick any odd number, so for the moment it's fine to just use 1. Why pick 1 over any other odd value? Well, it's the same train of thought as that "Computationally Easy Multipliers" stuff. The details vary by CPU, but usually an instruction set can encode a small value directly into an add instruction, without using a separate register. We're picking 1 so that *most of the time* we can skim another small boost out of it.

This gives us the next iteration of our LCG function:

```rust=
pub const fn lcg_u64_easy_step(x: u64) -> (u64, u64) {
    let next_state = x.wrapping_mul(0xAF251AF3B0F025B5).wrapping_add(1);
    let out = next_state;
    (next_state, out)
}
```

Let's see how we're doing on our generator criteria:

* Speed: blazing fast
* Size: 64-bit
* Period: `2**64`
* Quality: Bad
* Equidistribution: 1

That's a lot of work just to end up with a bad quality generator.

### Reducing Our Output Size

Okay so the problem is that with our `m` as a power of 2, every bit position will be on a `2**bit` cycle (numbering from 1). So the very top bit is on the full `2**64` cycle, and the lowest bit is on a `2**1` cycle, constantly flipping between 0 and 1. This makes a lot of our bits *very* predictable.

In practice, it's actually enough to hand out `u32` sized values instead of `u64` values. Say you need a random index into an array, there's probably less than `u32::MAX` elements.

So for each step we'll only give back the top 32 bits, and discard the rest.

```rust=
/// Steps an LCG using u64 values.
///
/// * `a` > 0
pub const fn lcg_u64_32_easy_step(x: u64) -> (u64, u32) {
    let next_state = x.wrapping_mul(0xAF251AF3B0F025B5).wrapping_add(1);
    let out = (x >> 32) as u32;
    (next_state, out)
}
```

Here's a fun trick: We determine the output value using the starting state instead of the final state to get just a tiny bit of [instruction level parallelism][wp-ilp], assuming your CPU can do that.

* Speed: blazing fast
* Size: 64-bit
* Period: `2**64`
* Quality: Not The Worst
* Equidistribution: 32

Hey that's an improvement! Now that we're only giving out the top 32 bits of the generator's state our quality has gone up a little bit, and the output is actually gonna be 32-dimensional equidistributed. Nice.

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

TODO

### Extending The Period

TODO

## Permuted Congruential Generators

TODO

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

<!-- Links -->

[wp-prng]: https://en.wikipedia.org/wiki/Pseudorandom_number_generator
[wp-lcg]: https://en.wikipedia.org/wiki/Linear_congruential_generator
[wp-pcg]: https://en.wikipedia.org/wiki/Permuted_congruential_generator
[wp-csprng]: https://en.wikipedia.org/wiki/Cryptographically-secure_pseudorandom_number_generator
[wp-xorshift]: https://en.wikipedia.org/wiki/Xorshift
[wp-ilp]: https://en.wikipedia.org/wiki/Instruction-level_parallelism
[github-TestU01]: https://github.com/umontreal-simul/TestU01-2009/
[sourceforge-PractRand]: http://pracrand.sourceforge.net/
[blubapedia-rng]: https://bulbapedia.bulbagarden.net/wiki/Pseudorandom_number_generation_in_Pok%C3%A9mon
[pit-of-success]: https://blog.codinghorror.com/falling-into-the-pit-of-success/
[how-to-test-with-TestU01]: https://www.pcg-random.org/posts/how-to-test-with-testu01.html
[osti-lcg-jump]: https://www.osti.gov/biblio/89100-random-number-generation-arbitrary-strides
[google-scholar-brown-94]: https://scholar.google.com/scholar_lookup?title=Random+number+generation+with+arbitrary+strides&publication_year=Sat+&author=F.+Brown&journal=Transactions+of+the+American+Nuclear+Society