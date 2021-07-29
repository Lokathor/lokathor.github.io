# A Little Bit About PRNG Stuff

Let's talk about [Pseudorandom Number Generators](https://en.wikipedia.org/wiki/Pseudorandom_number_generator), which are called PRNGs for short.

In general, they're a deterministic way to take some **state** and perform a **step** so that you'll have both a new state and also some random-seeming output value.

There's lots of possible PRNGs. There's even *families* of PRNGs, where they all use the same basic math and just a few constants are different (different multipliers, different offsets, different state size, etc).

Today we're going to be looking at PRNGs from specifically the perspective of implementing them in Rust code. I'm really not the best at math, but I am kinda accidentally good at programming, so I usually try to understand math *in terms of* what it means for some sort of programming thing. Then the math makes more sense to me.

Also, in this article we'll need to talk about "X to the power of Y" type values. In math, if you can't write actual superscript text, sometimes you'd write `X^Y`, but in Rust the `^` character means `bitwise_xor`. Since Rust doesn't naturally have any operator for the power function we'll go ahead and steal `**` from Python. So if we want "X to the power of Y" we'll write `X**Y`.

## Generator Criteria

There's several categories that we might care about when considering a PRNG.

First of all we can consider the **size** of the generator's state. This is very easy to quantify: how many bytes do we need for our struct? In general, we'd like all our structs to be as small as possible, and this is no different.

Also as usual for any programming thing, we also want the **speed** of the step function to be as fast as possible. This is also pretty easy to quantify, we can set up benchmarks and check the speed of different step functions for different generators. If we care about bulk randomness generation we might measure the output in terms of GB/seconds, which makes for a better comparison between generators that have different amounts of output per step.

The **period** of the generator is how many times you can step the generator before the output sequence loops around. Since they're deterministic, eventually *all* PRNGs will loop around. The period of a generator is *at most* equal to the number of possible bit states within the generator. However, not all generator state is always used with maximum efficiency towards having a long period, so sometimes the period is less than the number of possible bit states. Even if you don't plan to *actually* loop the generator, you usually want different runs of the program (or different threads within a single program) to not re-use parts of the sequence that you've gone over before, so a bigger period is always nicer to have. On modern machines, `2**64` should be the *minimum* acceptable generator period.

When we talk about the **quality** of the output generator we mean how random-seeming each output is. Is the output biased? Can the output be predicted based on the previous outputs? Stuff like that. Checking these things involves running the generator to get output and then doing statistical tests on that output. You're *not* expected to know how to do that yourself. Instead, there's test suites that are available that will run your generator and then do the statistics on the output and print out some results. [TestU01][github-TestU01] is a C library that has the `SmallCrush`, `Crush`, and `BigCrush` test batteries. [PractRand][sourceforge-PractRand] can be used as a library or as a CLI tool.

[github-TestU01]: https://github.com/umontreal-simul/TestU01-2009/
[sourceforge-PractRand]: http://pracrand.sourceforge.net/

A generator has **uniform** output if every possible output value appears an equal number of times in the full output sequence. It could be exactly once for each possible output, or it could be more than once, as long as all outputs appear an equal number of times. This might seem very simple, but some generators don't do this. If a generator is known to be uniform and you go through more than about half of the period, the apparent quality of the output begins to slowly drop, until the final output value becomes perfectly guessable. The logic goes like this: if all outputs showed up X many times, except one particular output which showed up X-1 times, the final output has to be the one that was X-1 times. If the period of your PRNG is large enough then this isn't really an actual problem in practice, but I think it's worth mentioning.

A concept that's similar to uniformity, but not quite the same, is **k-dimensional equidistribution**. This means "how many times *in a row* can you call the generator before the next output can't be any possible value in the output range?". If you're using multiple generator calls aggregated together then you probably really care about this one. Say you're generating 2D points, with one call for X and one for Y. You probably want to know if the point `(1,1)` can't ever *actually* occur because the output sequence doesn't ever have an output of `1` twice in a row. Maybe that chance of seeing a `(1,1)` is *very small*, but it should be *possible* right?

## My Kingdom For A Hardware Multiplier

The types of PRNGs we'll be focusing on today assume that your device is *at least* advanced enough to have efficient hardware integer multiplication. This might seem like a low bar, but lots of fun retro devices don't have that, or they do but it's costly. If you're programming for something where multiplication is a heavy cost and you want something cheaper than that check out the [XorShift][wp-xorshift] family of generators instead.

[wp-xorshift]: https://en.wikipedia.org/wiki/Xorshift

Because of the details of how the XorShift generator family works they never actually output 0, which means that they all have non-uniform output. I'd rather use generators that do have uniform output, so we won't be looking at the XorShift stuff today.

## The Generators We Will Talk About Here Are Not Cryptographically Secure

When a pseudorandom number generator meets some high enough standards to make it suitable for cryptography we call it a [Cryptographically-secure pseudorandom number generator][wp-csprng], or CSPRNG. This means that you can use the generator for keeping your secrets.

[wp-csprng]: https://en.wikipedia.org/wiki/Cryptographically-secure_pseudorandom_number_generator

The generators that we'll talk about here today **are not cryptographically secure**.

In other words, **do not use these generators for cryptography**.

I'm serious.

# Linear Congruential Generators

One of the easiest types of generator to understand and implement is the [Linear Congruential Generator][wp-lcg] (LCG). They're so simple to implement that they're used in countless programs. They're even used in pokemon!

[wp-lcg]: https://en.wikipedia.org/wiki/Linear_congruential_generator

The LCG family of generators was developed in 1958 by W. E. Thomson and A. Rotenberg, based on the slightly earlier Lehmer generator which had been developed by D. H. Lehmer in 1951. While there *are* some problems with this type of generator, they're well understood problems that we can compensate for.

## LCG Basics

The core formula of the LCG family is very simple. If you look at the wikipedia page they give it using the full fancy math notion, but the pseudocode version is:

```
// pseudocode

x_next = (a * x_current + c) `mod` m
```

The `mod` thing is something you might not be familiar with called [Modular Arithmetic][wp-mod]. Essentially the value of `a * x_current + c` is "wrapped" to be within 0 to `m`. Not only that, but there's some rules for all the parameters involved if we want it to work right:

[wp-mod]: https://en.wikipedia.org/wiki/Modular_arithmetic

* `m` > 0
* `a` > 0 && `a` < `m`
* `c` >= 0 && `c` < `m`
* `x_current` >= 0 && `x_current` < `m`

Hmm, so we could just use `%` to do a modulus division, but that's *way expensive* compared to the multiply and the add. Maybe we can like... get rid of the `mod`, or something? Math is usually flexible, you can rewrite expressions and stuff, it's all fine as long as you follow the correct rewrite rules. Do we got any rewrite rules that can help us? Let's check that wikipedia page. If we go to the "[Integers modulo n][wp-mod-integers]" section there's some stuff. There's even some rewrite rules we can try for addition, subtraction, or multiplication.

[wp-mod-integers]: https://en.wikipedia.org/wiki/Modular_arithmetic#Integers_modulo_n

```
// with OP as any of: addition, subtraction, or multiplication

(a OP b) `mod` m == ((a `mod` m) OP (b `mod` m)) `mod` m
```

Huh, well we've only got multiplication and addition in our LCG, and we've got rewrite rules for both of those operations. Let's try isolating all of our equation's components so that they're on their own.

```
// starting formula
x_next = (a * x_current + c) `mod` m

// break up the addition
x_next = (((a * x_current) `mod` m) + (c `mod` m)) `mod` m

// break up the multiplication
x_next = ((((a `mod` m) * (x_current `mod` m)) `mod` m) + (c `mod` m)) `mod` m
```

That's a *lot* of "`mod` m". Are we closer to something useful? All of our terms *and* all of our operations are "`mod` m" all the time. Know what else is "`mod` m" all of the time? If you said "unsigned intergers" you get a point! We can use unsigned integers and then have everything happen "`mod` m" in the hardware automatically for us.

If we use 32-bit integers then `m=2**32`, if we use 64-bit integers then `m=2**64`, and so on. Which means that we can pick an unsigned integer type and cancel out all the "`mod` m" parts, which simplifies our formula back down to:

```
// pseudocode

x_next = a.wrapping_mul(x_current).wrapping_add(c)
```

Convenient! I told you an LCG would be pretty easy to implement after all.

Okay, let's get some actual rust code going. We'll use `u64` values for everything, since that's the biggest we can efficiently go on a 64-bit machine. We'll also use const generics for the multiply and add value, since those values aren't supposed to change ever. We're gonna give them the name `MUL` and `ADD` though so it's more clear to others what they're doing. And the value that changes from step to step can be, uh, the `position`? When you step around you're changing your position, that makes sense. We don't want to just call it "state" since in a little bit we'll have some other state fields too.

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
#[repr(transparent)]
pub struct GenericLcg64<const MUL: u64, const ADD: u64> {
    position: u64,
}

impl<const MUL: u64, const ADD: u64> GenericLcg64<MUL, ADD> {
    pub fn next_u64(&mut self) -> u64 {
        self.position = self.position.wrapping_mul(MUL).wrapping_add(ADD);
        self.position
    }
}
```

Is our generator any good? Who knows! We can't even turn it on right now because we don't know what to put in for `MUL` and `ADD`.

## Selecting Our LCG Parameters

Let's go back to that Wikipedia page about how LCGs work. It says if we want the maximum period possible for our generator, and given that we've already selected `m=2**64`, then we need to have `ADD != 0` and we have to make sure that we satisfy the following rules (the "Hull–Dobell Theorem"):

* `m` and `ADD` are relatively prime (aka: `gcd(2**64, ADD) == 1`), which means `ADD` must be odd.
* `MUL-1` is divisible by all prime factors of `m`.
* `MUL-1` is divisible by 4, since `m` is divisible by 4.

That's still a lot of possible numbers. When in doubt, try to keep reading until it makes sense.

> Although the Hull–Dobell theorem provides maximum period, it is not sufficient to guarantee a *good* generator.

Flip. Okay, well, there's a chart of common parameters...

> This table is to show popularity, not examples to emulate; *many of these parameters are poor.*

Ferris, please save us.

> Tables of good parameters are available.\[2\]\[10\]

:3

Okay so let's check out those references numbered 2 and 10:

* 2: Steele, Guy; Vigna, Sebastiano (15 January 2020). "**Computationally easy, spectrally good multipliers for congruential pseudorandom number generators**". [arXiv:2001.05304](https://arxiv.org/abs/2001.05304) \[cs.DS\]. "At this point it is unlikely that the now-traditional names will be corrected." Mathematics of Computation (to appear). Associated data at <https://github.com/vigna/CPRNG>.
* 10: L'Ecuyer, Pierre (1999). "**Tables of Linear Congruential Generators of Different Sizes and Good Lattice Structure**". Mathematics of Computation. 68 (225): 249–260. [CiteSeerX 10.1.1.34.1024](https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.34.1024). [doi:10.1090/S0025-5718-99-00996-5](https://www.ams.org/journals/mcom/1999-68-225/S0025-5718-99-00996-5/home.html). Be sure to read the [Errata](https://www.iro.umontreal.ca/~lecuyer/myftp/papers/latrules99Errata.pdf) as well.

Ah, hmm, well... okay we check the most recent paper first. There's new numbers invented all the time, so we want recent numbers of course. Here's the [direct link to the PDF](https://arxiv.org/pdf/2001.05304.pdf) if you don't want to bother looking thought the arXiv website and finding the link yourself.

### Steele / Vigna 2020

When reading a paper for the first time, always take a moment to double check the Abstract and the Introduction to make sure that what you think it's talking about and what it thinks its talking about are the same thing.

> In this paper, we provide lists of multipliers for both MCGs and LCGs, continuing the line of work by L'Ecuyer in his classic paper.

Paydirt! Oh hey there's that github link... maybe it lists some numbers? Readme for the repo says their database of numbers is 14GB compressed, and 36GB decompressed. Sounds like the wrong avenue for us.

* *Chapter 2* talks about the Spectral Figures of Merit. Lattice, Hyperplanes, Spectral Test, Hermite Constant. I'm very lost among all these terms, but let's keep going. Surely we'll be able to tell a list of numbers when we see it.
* *Chapter 3* is about "Computationally Easy Multipliers". This is the interesting idea. If we're multiplying our position by a constant, then a smaller constant might take fewer instructions to get loaded into a register.
* *Chapter 4*: Bounds on Spectral Scores?
* *Chapter 5*: Beyond Spectral Scores? I hope someone understands this stuff, I don't.
* *Chapter 6* "Potency" gives us another way to determine if a multiplier is good or not. It also says that depending on our multiplier, the add value we pick will put our output sequence into one of several equivalence classes "up to an additive constant". I don't entirely know what that means, but I think it means once we pick a `MUL` value the `ADD` value is a lot less important, so we can just use 1 probably.
* *Chapter 7* is about MCGs, so that doesn't apply to us.

Ah, finally, *Chapter 8*: Tables. There's a wonderfully long explanation here of how all the different multipliers are ranked with spectral scores and harmonic scores and all sorts of things. The short version seems to be that, if we don't really know what we're doing:

* We can find the pages for the `m` value we're using.
* Make sure we're using the page that says `M+` at the top of the 3rd column (not `M*`, that's for MCG and we're doing LCG).
* Then pick a multiplier bit width, which can be less than your full LCG's bit width. As they mentioned in Chapter 3 of the paper, there is sometimes an advantage to picking a smaller bit width multiplier.
* Finally, if you only want one number, pick the 2nd value in each grouping of four numbers for that bit width.

So for an LCG with `u64`, we look in the chart and find `0xF691B575`. There's other values in the list we could also use, but we'll use that one for now. We finally have a `MUL` value!

The `ADD` value is much simpler.
We can pick any odd value, so we just need to make sure the lowest bit is always set.
We could do that with `ADD|1` of course.
There's a small problem though: when people pick a value "at random" they often enough pick a very small value where they'll notice that, say, 2 and 3 are the same stream, since 2|1 and 3|1 are the same.
Instead, we'll take a hint from how the PCG does it and use `(ADD<<1)|1`.
This throws out the highest bit of user input instead of the lowest, which they're probably less likely to notice.
We'll look more into the PCG stuff later on, this isn't the last trick we're gonna borrow from them.

```rust
impl<const MUL: u64, const ADD: u64> GenericLcg64_32<MUL, ADD> {
    pub fn next_u32(&mut self) -> u32 {
        let out = (self.position >> 32) as u32;
        self.position = self.position.wrapping_mul(MUL).wrapping_add((ADD<<1)|1);
        out
    }
}
```

Now we can also make an alias with "simple" parameters people can go by if they don't want to figure out their own.

```rust
pub type SimpleLcg64 = GenericLcg64<0xF691B575, 0>;
```

## Reducing The Output Size

Okay, so we've got a generator, but it's *not very good* right now.

The problem that we've got is that every bit in the generator's state is on a `2**bit` cycle, counting from 1 as the lowest bit. This means that the highest bit is on a `2**64` cycle, which is plenty large, but the lowest bit is on a `2**1` cycle. It's just constantly flipping between 0 and 1.

Alright, who cares? Well, the output of the generator at each step is the generator's entire state. This means that the predictable bits in the state become predictable bits in our output. Those top bits are fine, we just have an exponential fall off in quality as we go down the line of bits.

But, you know... in practice... well we don't need more than say 32 bits from a single step of the generator. Right? That's "big enough" nearly all the time. Let's just adjust the generator to only return `u32` values, and we can shift the upper half of the bits down and return those. Then we'd only be returning the good bits.

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
#[repr(transparent)]
pub struct GenericLcg64_32<const MUL: u64, const ADD: u64> {
    position: u64,
}

pub type SimpleLcg64_32 = GenericLcg64_32<0xF691B575, 1>;

impl<const MUL: u64, const ADD: u64> GenericLcg64_32<MUL, ADD> {
    pub fn next_u32(&mut self) -> u32 {
        let out = (self.position >> 32) as u32;
        self.position = self.position.wrapping_mul(MUL).wrapping_add(ADD);
        out
    }
}
```

Note that our output is formed from the position value *before* it changes, not after. This lets us get just a tiny hair of [instruction level parallelism][wp-ilp] going. This way computing the output doesn't depend on the change computation having completed. When the step function is this small it can help to shave off even one cycle.

[wp-ilp]: https://en.wikipedia.org/wiki/Instruction-level_parallelism

Also note that our generator has a new name! Before our generator was an "LCG64" type, because of a 64-bit position. Now that we're just keeping the best 32 bits it's known as an "LCG64/32". Of course there's plenty of possible multipliers and additives, so there isn't just exactly one LCG64/32, it's a whole family of potential generators.

## Testing With `SmallCrush`

Our generator isn't the best and can't pass any of the intense test suites, but at this point it should be able to at least pass the `SmallCrush` test suite from `TestU01`.

Thankfully, there's already a blog post on [how to test with TestU01][how-to-test-with-TestU01], so we'll mostly go by that. One minor problem: TestU01 is a C library. I don't want to bother with making it callable from Rust and all that, so for our quick test we'll just adapt the sample C program in the blog post into one that runs our LCG64/32. I'm not really a C programmer, but I know enough to do this little bit.

[how-to-test-with-TestU01]: https://www.pcg-random.org/posts/how-to-test-with-testu01.html

```c
// adapted from https://www.pcg-random.org/posts/how-to-test-with-testu01.html

#include "TestU01.h"
#include <stdint.h>

static uint64_t y = 2456;

uint32_t lcg64() {
  uint32_t out = (uint32_t)(y>>32);
  y = y * 0xF691B575 + 1;
  return out;
}

int main() {
  unif01_Gen *gen = unif01_CreateExternGenBits("lcg64 blog", lcg64);
  bbattery_SmallCrush(gen);
  return 0;
}
```

Okay, now we build the program from within our `/TestU01` directory that we have from following the blog post instructions on how to get the library set up:

```sh
gcc -std=c99 -Wall -O3 -o lcg64 lcg64.c -Iinclude -Llib -ltestu01 -lprobdist -lmylib -lm
```

And finally we run the thing. A whole ton of stuff spits out, and then at the end we see

```
========= Summary results of SmallCrush =========

 Version:          TestU01 1.2.3
 Generator:        lcg64 blog
 Number of statistics:  15
 Total CPU time:   00:00:07.65

 All tests were passed
```

Awesome!

An LCG64/32 can't pass the stronger test batteries in TestU01. It's just fundamentally not a good enough generator, and there's not enough internal state to hide the problems. If we had an LCG128/32 that could pass (because of the additional state), but it would also be notably slower (128-bit ops must be emulated on 64-bit machines).

For now, passing `SmallCrush` is good enough, and we'll switch over to talking about some fun additional things we can let our LCG do.

## Jumping The Generator Position

Here's the first fun special technique that we can do with our LCG.
If we want to send the generator forward by `delta` steps, without actually calling the generator's step function, we can!
I don't mean we're gonna just do `delta` multiplies all in a single method, I mean we'll go `delta` steps in less than `delta` time.
It's described in a paper called "**Random Number Generation with Arbitrary Stride**".
If we look up that paper name we can get ourselves to the [OSTI page for the paper][osti-lcg-jump], which... I guess we click [find in google scholar][google-scholar-brown-94]?
Right and... okay here's a [direct PDF Link](https://mcnp.lanl.gov/pdf_files/anl-rn-arb-stride.pdf), finally.

[osti-lcg-jump]: https://www.osti.gov/biblio/89100-random-number-generation-arbitrary-strides
[google-scholar-brown-94]: https://scholar.google.com/scholar_lookup?title=Random+number+generation+with+arbitrary+strides&publication_year=Sat+&author=F.+Brown&journal=Transactions+of+the+American+Nuclear+Society

Let's have a look.
Hmm, the paper is showing the same formula as before, but they're calling the multiplier `g` instead of `a`.
Other than the different name it's the same equation.
Always helps to know you're on the same page.

Next we've got a formula to directly jump a generator `k` steps forward:

```
s_(k) = (s_(0) * g**k + c * (g**k - 1)/(g-1)) `mod` 2**m
```

However, there's a mild snag.
More like a big snag.
This formula assumes *actual* integers.
You know, real integers like math uses, that don't have bit limits.
No wrapping values to stay within a specific bit width.
To not accidentally run out of bits when using this formula, we'd need to do all the math at *twice* the bit width as our LCG position.
With a `u64` position, we could maybe do `u128` math.
If we wanted to scale things up and have a `u128` position, well Rust doesn't even have a `u256` yet.
There's gonna be some better way, right?

Yeah, if we just turn to the next page we see some lovely follow-up formulas:

```
G = g**k `mod` 2**m
  = some sort of summation `mod` 2**m
  = some sort of product `mod` 2**m
  // I don't know how to write those ops out well,
  // if you wanna know the details look in the paper

C = c * (g**k - 1)/(g-1)) `mod` 2**m
  = c * (1 + g + g**2 + ... + g**(k-1)) `mod` 2**m
```

What?
How does this help?
Well the paper proposes that we rewrite things like so:

```
// before

s_(k) = (s_(0) * g**k + c * (g**k - 1)/(g-1)) `mod` 2**m

// after

G = ...
C = ...
s_(k) = (s_(0) * G + C) `mod` 2**m
```

So we'll compute a value for `G`, which we'll lovingly call "big G", and also a value for `C`, "big C".
Both big G and big C will be normal `u64` values.
Then the final operation becomes a multiply and an add like we've already been doing.

We even get some pseudocode on precisely how to compute big G and big C while staying in the limits of common programming languages:

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

I'm sure you're wondering, "what's up with that `i <- (k + 2**m) `mod` 2**m`? isn't that a no-op?"
The answer is that it's sorta trying to say that this works if you want to "go negative steps".
That is, if you want to go to a previous state of the generator.
Makes sense, since if you go far enough forward the generator output loops around eventually.

If you look at all that closely you'll see that Algorithm G and Algorithm C share many common elements.
We can merge the two loops as we convert it to Rust.
The code is something like this:

```rust
impl<const MUL: u64, const ADD: u64> GenericLcg64_32<MUL, ADD> {
    pub fn jump_position(&mut self, delta: u64) {
        let mut big_g = 1_u64;
        let mut h = MUL;
        let mut big_c = 0_u64;
        let mut f = (ADD<<1)|1;
        let mut i = delta;
        while i > 0 {
            if (i & 1) != 0 {
                big_g = big_g.wrapping_mul(h);
                big_c = big_c.wrapping_mul(h).wrapping_add(f);
            }
            f = f.wrapping_mul(h.wrapping_add(1));
            h = h.wrapping_mul(h);
            i /= 2;
        }
        self.position = self.position.wrapping_mul(big_g).wrapping_add(big_c);
    }
}
```

That's a lot of stuff, we could have gotten something wrong pretty easily.
Let's add a test case too just to see if things work like we think.

```rust
#[test]
fn test_lcg_u64_jump() {
    let g = StandardLcg64_32 { position: 5678 };

    let mut g_jumped1 = g.clone();
    g_jumped1.jump_position(1);
    let mut g_next1 = g.clone();
    g_next1.next_u32();
    assert_eq!(g_jumped1.position, g_next1.position);

    let mut next_back_one = g_next1.clone();
    next_back_one.jump_position(-1_i64 as u64);
    assert_eq!(g.position, next_back_one.position);

    let mut g_jumped50 = g.clone();
    g_jumped50.jump_position(50);
    let mut g_next50 = g.clone();
    for _ in 0..50 {
        g_next50.next_u32();
    }
    assert_eq!(g_jumped50.position, g_next50.position);
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=28aac9cb4a1e4f05b2ca1c7e8d2bac94)

## Multiple Streams

When you change the additive value it changes the ordering of all the outputs
If we wanted to have a little more possible variety we could have the `ADD` parameter be a field in the struct.
That way the user could select which stream of numbers they were going through at runtime instead of always using an add of 1.

```rust
pub struct MultiStream_GenericLcg64_32<const MUL: u64> {
    position: u64,
    stream: u64,
}
```

Nothing much changes with this multi-stream generator.
Anywhere we'd use `(ADD<<1)|1` we instead use `self.stream`.
We can adjust the `stream` value to ensure that it's odd just once in the generator's constructor,
then we don't have to repeat the shift and bitor every time we use it.

It's a fairly obvious transformation to go from a const generic to having the value as a field in the struct.
Still, you might want to consider it if you're looking for a little more runtime variety without too much additional code complexity.

The downside of this generator augmentation is that since the `position` is changing, but our `stream` doesn't change, we've doubled our required *state* without increasing our generator's *period*.
Then again, maybe you just want more possible initial states for your program and you know it won't run through the whole period anyway.

## Digit Counter Generator

This is the first of two ways we can get `k`-dimensional equidistribution on our generator.
Both of these are both described by M.E. O'Neill in the [PCG paper][pcg-paper].
One of them is described as being an LCG technique, and one is listed in the PCG section.
Really, both of the extension techniques can be used by either type of generator.

[pcg-paper]: https://www.pcg-random.org/pdf/hmc-cs-2014-0905.pdf

This technique we'll call a "Digit Counter" generator.
To get our `k`-dimensional generator, we'll have `k` inner generators all in operation.
When the "lower digit" generators roll over they'll "carry" and cause the next generator up to step an extra time.

Imagine we have a [counter clicker](https://www.alfaplanhold.com/v/vspfiles/photos/9241000-2.jpg?v-cache=1609401859) device. 
It's got a button, you push the button and the 1s digit goes up. When that rolls over from 9 to 0 then it carries and the 10s digit goes up.
The pattern repeats up through all the digits, just that the highest digit doesn't carry into anything else.
Eventually you'll see every possible digit combination, if you click through the entire sequence.
So we know that if we have `k` different digit slots, then the overall sequence is `k`-dimensionally equidistributed.

A problem is that when we look at each block of output from the clicker, most of the digits are the same.
Most of the time only change one out of all the `k` different digits: `000`, `001`, `002`, etc.
That's not very *random seeming*, is it?
What we'll do instead is advance *every* digit with *every* click, in addition the carry when a digit rolls over.
So with "advance the 1s digit" we'd have `008`, `009`, `010` (carry), `011`.
With "advance all digits" we'd have `008`, `119`, `230` (carry), `341`.
You might think that each individual digit is still fairly predictable,
but when we use *real* generators with this system the "advance a digit" step will become much more unpredictable, instead of just "+1".

Now, one thing here is that if each generator is *actually* an identical sequence, then sometimes the overall collection of generators will end up closely in phase.
Imagine if our clicker counter got to `111`.
The next step would be `222`, and `333`, until eventually `999` became `110` and they broke out of lockstep.
That'll make the output temporarily not so great.
To prevent this from happening we'll use a separate add value for each digit.
Occasionally all the digits will show the same value, but since the sequences are all differently ordered they'd quickly jump back out of the lockstep.

Exactly how you have a different add value per generator digit is up to you.

* If we want to have just one add value for the whole thing we can just add +2 to the add value as we go to each next generator.
* Alternately we could have a completely distinct add value *per generator digit*, if we wanted to really bump up that possibility space.

Also, this generator works best for *block* output.
If you're outputting just one value at a time you need to store which "digit" you advanced last.
Storing an index of "which digit is next" takes up state space which aren't actually helping the generator in any category we care about.
If we output an entire block of values at once we never store which digit is next, since every call touches every digit.

Let's finally get to some code.
For this generator we'll have a method that takes a reference to an array and fills it in with new values.
That saves us a lot of accidental stack copying whenever `next_block` doesn't get inlined.
Also for this exact example we'll go with the "one basic `add` value that we adjust automatically between the digits" style.

```rust
// note: can't derive Default, we'd have to write the impl ourselves.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
pub struct DigitCounter_GenericLcg64_32<const MUL: u64, const ADD: u64, const DIGITS: usize> {
    digits: [u64; DIGITS],
}

impl<const MUL: u64, const ADD: u64, const DIGITS: usize>
    DigitCounter_GenericLcg64_32<MUL, ADD, DIGITS>
{
    /// Fills the array reference given with the next block of outputs.
    pub fn next_block(&mut self, target: &mut [u32; DIGITS]) {
        // gather output based on the *previous* step's results like we normally do.
        target
            .iter_mut()
            .zip(self.digits.iter())
            .for_each(|(t, d)| {
                *t = (d >> 32) as u32;
            });
        // we just adjust a single add value as we go.
        let mut add = (ADD << 1) | 1;
        // the lowest digits always updates exactly once
        self.digits[0] = self.digits[0].wrapping_mul(MUL).wrapping_add(add);
        for i in 1..DIGITS {
            add += 2;
            self.digits[i] = self.digits[i].wrapping_mul(MUL).wrapping_add(add);
            // higher digits do an extra add if the previous digit landed on 0.
            if self.digits[i - 1] == 0 {
                self.digits[i] = self.digits[i].wrapping_add(add);
            }
        }
    }
}
```

In addition to having the `k`-dimensional equidistribution that we were after, this generator will have a *very* large period.
If each internal "digit" is a `b`-bit LCG (period of `2**b`), our combined `k`-digit counter generator has a period of `2**(b*k)`.
This is a "perfectly efficient" generator period period, where our period is 100% of our state bits.

Also, the normal formula for jumping an LCG's position doesn't work with this digit generator.
The formula doesn't have any way to account for the carry interaction from one digit to the next.
Then again, if you don't care about it being exactly perfect, you could still run the jump operation on any or all of the digits to just get "somewhere else far away in the sequence".

## Xor Array Generator

Having a Digit Counter Generator with block output is fine sometimes.
On the other hand, we might want a generator that has normal "one value at a time" output,
and just *happens* to also provide `k`-dimensional equidistribution as well.

Like I said, the PCG paper has a second extension scheme we can use for this situation.

This time our generator will be fundamentally "one" generator, with only one `position` value.
However, we will add an "extension array" which holds elements of our generator's output type.
The size of the extension array should be a power of 2, you'll see why in a moment.

```rust
/// For unbiased results, `K` should be a power of 2.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
pub struct ExtensionArray_GenericLcg64_32<const MUL: u64, const ADD: u64, const K: usize> {
    position: u64,
    ext: [u32; K],
}
```

Now every time we would output a value from the generator we're going to `xor` the value with an element from the extension array.
We'll select which extension array slot at random by using bits from our `position`.
If we want the slot usage to be very predictable we can use the lowest bits, otherwise we can use the highest bits.
This is why `K` should be a power of 2.
When that's the case, it's easy to use the highest or lowest bits from `position` to make an unbiased index into the extension array.

So far it's kinda like we've just mixed `K` different output sequences together.
One per extension array slot, right?
If we `xor` the intermediate output value with an extension array value, that obviously changes the output.
If we were to *always* `xor` with the same extension array slot, then we'd have a standard output sequence changed to some new ordering of values.
When we select different extension array slots per output, it's like we're jumping between several different output sequences.

That's not enough for `k`-dimensional equidistribution.

The final thing we need is a way to "advance" the extension array at some regular interval.
We must advance the extension array *at least* once per `position` period, and we could advance it more often if we like as long as the interval is regular.
We also need our "advance" operation to be something that causes the array to go through a full period.

Since we already talked about the digit generator, we can use a system like that for advancing the array.
This time we can just use "+1" as the "per digit" advancement system for simplicity.

```rust
impl<const MUL: u64, const ADD: u64, const K: usize> DigitCounter_GenericLcg64_32<MUL, ADD, K> {
    pub fn next_u32(&mut self) -> u32 {
        // mask with an extension array slot
        let out = (self.position >> 32) as u32 ^ self.ext[self.position % K];
        // if our position is 0 we advance the array
        if self.position == 0 {
            let mut last_carried = false;
            self.ext.iter_mut().for_each(|x_mut| {
                let (new, carried) = x_mut.overflowing_add(1 + (last_carried as u64));
                *x_mut = new;
                last_carried = carried;
            });
        }
        // advance the position
        self.position = self.position.wrapping_mul(MUL).wrapping_add((ADD << 1) | 1);
        out
    }
}
```

The disadvantage of using "+1 per slot, with carry" as the extension array advancement scheme is that each extension array value doesn't change much per array step.
However, it does let us jump forward and back by X many base periods by just adding X or subtracting X (with carry) from all array slots.
Having a way to cheaply jump around is kinda neat.
If you don't care about being able to cheaply jump, and in easy multiples, you can use an alternate update scheme.
For example, we could add 1431655765 instead.
Alternately written as `0x55555555` or `0b01010101_01010101_01010101_01010101`, it'll cause many more bits to change compared to adding 1 (at least on average).
Or maybe you don't even add a value at all.
You could run some permutation function on the bits.
Like I said, any way of updating the array is fine as long as the array update scheme lets the array have a full period cycle.

The Xor Array generator has an absurdly large period, the same as the Digit Counter generator.
With `b`-bit LCG base generators, giving `r`-bit output, and `k`-equidistribution, we get a total period of `2**(b+(r*k))`.
As with the Digit Counter Generator, this means our period is 100% of our state bits.

# Permuted Congruential Generators

We've seen many things we can do with an LCG, and I've mentioned a thing called a "PCG" too.
The initials PCG stand for Permuted Congruential Generator, and you can think of it as "an LCG but better".

The problem that we have with the LCG is that the bit quality drops drastically the lower we go down the line of bits.
Our basic answer to the problem was "only keep the top half of the bits".
The PCG family uses the idea that if we apply a "permutation" to the raw LCG output we can get a much better output quality boost than a plain shift gives.

## Permutations

If you go and read the full [PCG paper][pcg-paper] you'll see that there isn't just one permutation function provided.
Instead, there's several suggested permutation building blocks that you can mix together.

All the building blocks have fancy initials for short because you're expected to combine more than one of them together.

* **Random Shift (RS):** We use the top bits (the best) to determine how much to shift the next (still mostly good) bit by, then keep those.
* **Random Rotation (RR):** Similar to the shift, but we use a randomized rotation instead of a shift.
* **Xor Shift (XSH):** Performs an "xorshift" by a constant (`x ^= x >> CONST;`)
* **Xor Shift Low (XSL):** Xor the top half of the bits down by half and xor with the bottom half of the bits. This is particularly efficient for `u128` on 64-bit devices because the two halves of the value are already stored in two `u64` registers, and the shift itself is skipped.
* **Random Xor Shift (RXS):** Xorshift, but by a random amount instead of a constant.
* **Multiply (M):** Multiply by a good quality MCG constant (you can get one from that Steele / Vigna paper).

The combined permutation has a big jumble of letters for a name that says what building blocks are used.
Here are some examples:

```rust
// half size output

pub const fn xsh_rr_u64_u32(x: u64) -> u32 {
    let temp: u64 = x ^ (x >> 18);
    ((temp >> 27) as u32).rotate_right((x >> 59) as u32)
}

pub const fn xsh_rs_u64_u32(x: u64) -> u32 {
    let temp: u64 = x ^ (x >> 22);
    (temp >> (29 - (x >> 61))) as u32
}

pub const fn xsl_rr_u128_u64(x: u128) -> u64 {
    ((x as u64) ^ ((x >> 64) as u64)).rotate_right((x >> 122) as u32)
}

// full size output

pub const fn rxs_m_xs_u32(x: u32) -> u32 {
    let mut temp: u32 = x ^ (x >> (4 + (x >> 28)));
    temp = temp.wrapping_mul(277803737);
    temp ^ (temp >> 22)
}

pub const fn rxs_m_xs_u64(x: u64) -> u64 {
    let mut temp: u64 = x ^ (x >> (5 + (x >> 59)));
    temp = temp.wrapping_mul(12605985483714917081);
    temp ^ (temp >> 43)
}

pub const fn xsl_rr_rr_u128(x: u128) -> u128 {
    let low64 = ((x as u64) ^ ((x >> 64) as u64)).rotate_right((x >> 122) as u32);
    let high64 = ((x >> 64) as u64).rotate_right(low64 as u32 & 63);
    ((high64 as u128) << 64) | (low64 as u128)
}
```

So instead of doing a left shift and truncate, we'd do one of these.

## Building A Pcg

Actually putting one of these PCG things together is 99% the same as how an LCG works.

The only difference is that instead of doing `(self.position >> 32) as u32` to get our output,
we call whichever permutation function we want to use.

The general outline is like this:

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
#[repr(transparent)]
pub struct GenericPcg32_PERMUTATION_FN<const MUL: u64, const ADD: u64> {
    position: u64,
}
impl<const MUL: u64, const ADD: u64> GenericPcg32_PERMUTATION_FN<MUL, ADD> {
    /// Produces the next 32-bits of pseudorandom output.
    pub fn next_u32(&mut self) -> u32 {
        let out = PERMUTATION_FUNCTION(self.position);
        self.position = self.position.wrapping_mul(MUL).wrapping_add((ADD << 1) | 1);
        out
    }
}
```

Because the PCG is just applying an alternate transformation to the output, all of those tricks we saw before about jumping the position and extending the generator and all that can still be used here.

One note on the naming:
* An LCG's number is based on the core state bits: LCG64 when you've got a 64-bit core state value that gets multiplied and added to.
  If the out is the highest X bits you put that after a slash.
* With a PCG, instead the main number that's shown is the *output* bits.
  After the `PcgXX` part you also put the name of the permutation used, if you're being very specific about it.
  From there, the size of the state can be inferred based on the permutation used.
  So if you see `Pcg32_Xsh_Rr` then that's going to have a 64-bit state.
  The output is 32-bit, and the "xsh-rr" permutation is a half-sized output permutation.

<!--

TODO

## Testing with BigCrush

TODO

## Testing With PractRand

TODO

# Generating Bounded Integers

TODO

# Generating Normalized Floats

TODO

https://allendowney.com/research/rand/downey07randfloat.pdf

# Seeding Your Generator

TODO

-->