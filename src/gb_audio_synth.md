
Recently a thing called [GB Studio](https://www.gbstudio.dev/) came out that
revived a bit of public interest in the Game Boy as a system. It's neat. Like
having RPG Maker for the Game Boy. You probably couldn't use it to make a super
high complexity Game Boy game, the Game Boy hardware is really just _so_ limited
that you need to be programming in ASM. Still, for beginners who just want to
try things out it's always better to have it than not have it.

Well, in a similar spirit, what if we wanted to reproduce some elements of the
Game Boy experience without living by the _actual_ Game Boy hardware limits?
Something like Shovel Knight, or Sonic Mania, where you start with the look and
sound of an old console game, and then [break past the real
limit](https://yachtclubgames.com/2014/07/breaking-the-nes/) of what the
hardware at the time could really do in just a few ways.

# Game Boy Audio Synthesis

We're gonna be making some sounds that sound like the sounds the Game Boy made.

We'll be making a system that has a "close to authentic" output, but without
following the exact Game Boy limits, and one that is easy to control in the
style of a modern library. It obviously will be brazenly wasteful, using entire
_bytes_ to store a number when just 2 bits could have done the job, things like
that. Yet, despite this, I think it will turn out great.

* **Just to be clear from the start:** this will _not_ end with a system ready
  made for use as part of of a Game Boy emulator. We won't bother to follow the
  exact memory layout that the Game Boy had, and we won't bother to be accurate
  down to the cycle with our sounds. If you'd like to actually make an emulator
  I think this will still be a helpful explanation to read, but you'll have to
  be stricter about how you implement things in your emulator's version.

I will be writing the code samples in Rust, but the theory is the same for any
programming language, so hopefully you can read this even if you don't know
Rust. I will attempt to keep my use of any Rust magic to a relative minimum. I
will, for example, leave off the `derive` attributes and such from the sample
struct definitions, because they're usually not critical. If you want to see the
full code with every gorey detail I'll give a link at the end.

## Previous Work

All the information here about how sound works on the Game Boy comes from a few
key sources that I want to give credit to before we proceed:

* [The Ultimate Game Boy
  Talk](https://www.youtube.com/watch?v=HyzD8pNlpwI&t=24m7s) (the time code in
  the link is queued to the sound section of the talk)
* [Pan Docs: Sound
  Controller](http://gbdev.gg8.se/wiki/articles/Sound_Controller)
* [Game Boy Sound
  Hardware](http://gbdev.gg8.se/wiki/articles/Gameboy_sound_hardware)
* [The Awesome Game Boy Development
  GitHub](https://github.com/gbdev/awesome-gbdev), and the members of their
  [gbdev Discord](https://discord.gg/gpBxq85)

I **don't** expect you to have read all those articles, but I **do** expect you
to have watched the sound portion of _The Ultimate Game Boy Talk_ at least once
(it's only about 2 minutes).

## Some Terms

Just a few terms before we go on:

* **DMG:** The Game Boy's production identifier was DMG for "Dot-Matrix Game".
  I've been trying to not type "DMG" before this point in the document and it's
  already been a bit hard. I'll probably say DMG all over the place by the time
  we're done.
* **Channel:** This word sometimes gets used to describe different kinds of
  things. I am personally using SDL2 for audio output in my "scratch project"
  where I've built this and other things, and you might be using SDL2 yourself
  as well. They use "[channel](https://wiki.libsdl.org/SDL_AudioSpec)" to mean
  the number of output locations (eg: mono, stereo, quad, or 5.1) for the sound
  stream, so we'll also use "channel" to mean the same thing. The Game Boy
  couldn't actually produce different sounds in the Left and Right channels, but
  you could adjust the volume level for each side separately, so we'll have a
  "slightly" stereo sound output.
* **Sample:** This is the smallest moment of sound output, like a pixel for
  sound. It includes one value for each channel at that moment. You can probably
  find at least one thing that uses each different number primitive type as a
  sample format, but we'll be using [signed 16-bit
  integers](https://en.wikipedia.org/wiki/Audio_bit_depth) (`i16` in Rust
  terms), one of the most common formats. Very little of our math will be
  particular to using `i16`, and you should be able to adapt the system to any
  other format if you need. The sample value itself represents how far forward
  or back the speaker should position [its
  magnet](https://en.wikipedia.org/wiki/Loudspeaker).
* **Frequency:** A frequency is some number of cycles per second. A frequency
  value is measured in hertz (Hz): 1 Hz is one cycle per second.
* **Samples Per Second / Sample Rate:** If a sample is like a pixel for sound,
  your samples _per second_ is like your pixels per inch in a picture. Just like
  how more pixels per inch gives a clearer image, more samples per second gives
  a clearer sound. There are [many common
  frequencies](https://en.wikipedia.org/wiki/Sampling_(signal_processing)#Sampling_rate).
  You don't want to go overboard if the situation doesn't call for it: the down
  side to having more samples per second is that it takes more time to compute
  the sound and it takes more memory to store. We'll use 48,000 samples per
  second as our target output rate, which is the "basic" level of professional
  quality.
* **Voice:** In _The Ultimate Game Boy Talk_ the term
  "[voice](https://www.rolandcorp.com.au/blog/a-to-z-synthesizer#SGVoice)" is
  used to mean each of the four types of sounds that a Game Boy could make.
  That's a common enough term in synthesizer stuff, so we'll just stick with
  calling each type of sound a "voice": Pulse A, Pulse B, Wave, and Noise.

## Our Output Target

Obviously there's many sound APIs available, but they all fundamentally involve
filling a buffer that's calibrated to a given samples per second with sound
samples. The details of when our synthesizer gets called with a buffer to fill
out aren't too important, we'll just assume that we're given a playback speed
and a buffer to fill and we will fill out as much buffer as we're given.

I'm using SDL2 for audio, and SDL2's
[AudioSpec](https://wiki.libsdl.org/SDL_AudioSpec) type expects stereo samples
to be stored in "LRLRLR..." ordering, so that's what we'll be doing. Let's make
a struct for this so that we can clearly label the left and right part without
having to remember which is which. The `repr(C)` part here is a slightly dumb
low level detail that just means it's guaranteed to be laid out in memory just
as C would do it, in order of the fields named.

```rust
/// A single moment of stereo sound.
#[repr(C)]
pub struct StereoI16 {
  pub left: i16,
  pub right: i16,
}
```

If you're targeting some other sound API then you can swap around the field
ordering, change to another data format, whatever you need to do. The rest of
our code will be done in terms of samples in general, so it should be easy
enough to change the details of a sample if you really need to.

Okay, now let's look at what the top-level function signature for our sound
output might be:

```rust
/// Uses the Audio Processing Unit (APU) config to fill the sound buffer.
///
/// Naturally, this changes the state of the APU, as timers tick up, reset,
/// change various fields, etc.
///
/// The buffer passed in is assumed to have been zeroed ahead of time, if
/// necessary.
pub fn synth_sound(config: &mut APU, rate: i32, buffer: &mut [StereoI16]) {
  unimplemented!("TODO: all of it")
}
```

## Audio Processing Unit (APU) Basics

The Audio Processing Unit, which emits all the sound data, is called the APU for
short. We definitely want a struct for this, it sounds very important to the
whole sound process. What's inside our APU struct? Well we can start with what
the DMG really has for sound registers and then add more if we need to.

Obviously there's the four voices, which we'll get to in a moment, but there's
also some other registers that aren't part of a voice which we should consider:

* **FF24/NR50:** This controls the levels for Sound Out 2 (left) and Sound Out 1
  (right), as well as if the Vin should go out to the left and right. No game
  ever used the Vin, so we won't consider it. The volume levels range from 0 to
  7, so we'll use a byte for that per channel.

```rust
pub struct APU {
  pub left_volume: u8,
  pub right_volume: u8,
}
```

* **FF25/NR51:** This register has flags for if a given voice should output to a
  particular side or not. This is an area where we can break past the real
  limits a bit, so we'll just store one volume level byte per channel per
  voice.

```rust
pub struct APU {
  pub left_volumes: [u8; 4],
  pub right_volumes: [u8; 4],
}
```

* **FF26/NR52:** Finally there's a register where we can check if a sound is on
  and enable or disable the entire sound system. On a real Game Boy you get like
  13% more battery life if sound is entirely off like that because the sound
  hardware uses power to run even if volume levels are 0. In our software
  synthesizer we can just _not call the function_ if we want to skip sound
  processing, so we won't bother to model this part.

Okay, now we'll add one field for each voice. Except, when you're setting the
configuration for a particular voice, you probably want the volume level to be a
part of that voice's info. So we'll take the volume fields out of the `APU`
struct and put them into the struct for each particular voice. Also we'll derive
some traits for our struct.

```rust
pub struct APU {
  // empty
}
```

Great! No code obviously means no bugs! And fast compiles.

## Voice Basics

Before we jump into handling any specific voice, we have a lot to cover about
how to process a voice in general.

Everything related to DMG sound happens at some frequency.

* Every voice's wave form has a highly variable frequency. Depending on the
  voice, this can go from between 32 Hz up to 524,288 Hz. Usual frequencies for
  "music" sorts of things are likely to be 65 Hz (C-2) up through maybe 2,093 Hz
  (C-7), but sound effects might easily go higher.
* The "Sweep" effect (Pulse A), if active, triggers in N * 1/128th of a second
  intervals, where N can be 1 through 7.
* The "Envelope" effect (Pulse A, Pulse B, and Noise), if active, triggers in
  N * 1/64th of a second intervals, where again N can be 1 through 7.
* The "Length" effect (all voices), if set, will automatically stops a voice
  after N * 1/256th of a second. In a real DMG the maximum Length you can set
  depends on the voice, but we'll just let all voice timeouts go up to 255.

We need to keep track of how time is passing as we're filling out the buffer so
that we can advance through the wave forms at the appropriate frequency, and
also so that we can trigger the effects at the appropriate times. Time "passes"
every time we fill in one sample of the buffer. Time can't pass in less than one
sample worth of time.

Our sound waves are discrete, not continuous. This means that instead of being a
mathematically described wave form that's precise out to infinite precision,
we've got a very definite amount of precision. In fact our waves are ultimately
just like the sound buffer we're filling: a series of sample "**chunks**", and
the wave form's output frequency is like the samples per second. We'll call them
chunks instead of also calling them samples so that it's clear when we're
talking about sound source and when we're talking about sound destination. The
way that we determine the output sample value depends on the output frequency of
the wave form (in comparison to our 48k samples per second):

* **Low Frequency** wave forms will advance _one or less_ chunks per sample,
  which makes it easy to process. We advance a bit, check the value of the
  current chunk, and that's our value for that sample. Easy.
* **High Frequency** wave forms will advance _more than one_ chunk per sample,
  and then we have to average together the value of all the chunks we cross over
  during that sample. It's a lot more fiddly.

The exact break point of where "low" and "high" frequency are depends on the
number of chunks in the wave form.

* The Wave voice has 32 chunks, provided by the user.
* The Pulse A and Pulse B voices are described in terms of off and on time
  percentage: 12.5%, 25%, 50%, or 75% (which sounds the same as 25%). We need a
  minimum of 8 chunks to model all possible settings properly.
* The Noise voice is somewhat different. Instead of storing a wave form like
  Wave does, or having a very simple formula like the Pulse voices do, there's a
  pseudo-random number generator that, depending on a setting, has either a 127
  steps in the sequence or 32,767 steps in the sequence. Also, **unlike the
  other voices**, the "frequency" of the Noise channel doesn't indicate how
  often we do a _full loop_ of the sequence, it instead indicates how often we
  _do one step_ of the pseudo-random sequence. If we treat it as having a
  "chunks / cycle" value of 1 then it'll kinda fit into the rest of our math.

What's the exact formula to find the high/low break point? We have to go back to
basic algebra / physics sorts of math for this one. There's many figures here,
each of them has an associated dimensional units, and we must track how the
units change as we multiply and divide different figures by each other.

Here's our base terms:

| Term | Dimensions |
|:-----|:-----------|
| sample_rate | samples / second |
| frequency | cycles / second |
| wave_chunks | chunks / cycle |

Remember that when you divide one fraction by another fraction it's the same as
multiplying the first fraction by the _inverse_ of the second fraction:

* `(a/b) / (c/d) == (a/b) * (d/c)`

So, if we combine some of those figures, these are how the dimensions change.
The steps are written out to try and help you follow what's going on, because
changing your dimensional units around is always a little tricky. Since all our
stuff happens in terms of samples emitted, we're aiming to find out how we can
compute `chunks / sample`.

| Term | Dimensions |
|:-----|:-----------|
| sample_rate / frequency | (samples / second) / (cycles / second) |
| sample_rate / frequency | (samples / second) * (seconds / cycle) |
| sample_rate / frequency | samples / cycle |
| (sample_rate / frequency) / wave_chunks | (samples / cycle) / (chunks / cycle) |
| (sample_rate / frequency) / wave_chunks | (samples / cycle) * (cycles / chunk) |
| (sample_rate / frequency) / wave_chunks | samples / chunk |
| 1 / ((sample_rate / frequency) / wave_chunks) | 1 / (samples / chunk) |
| 1 / ((sample_rate / frequency) / wave_chunks) | chunks / sample |
| 1 * (wave_chunks / (sample_rate / frequency)) | chunks / sample |
| wave_chunks / (sample_rate / frequency) | chunks / sample |

* If there's more than one chunk per sample we have to average all the chunks in
  that sample period together when computing the sample.
* If there's one or less chunks per sample, each sample value comes from just a
  single chunk.
  
If you wanted to be _even more accurate_ with your chunk sampling you could
track when a partial chunk crossed a chunk bound and
[lerp](https://en.wikipedia.org/wiki/Linear_interpolation) between them. For
example, if your chunks per sample is 0.4, then your chunk positions for each
sample would be at positions 0.4, 0.8, and then when you move to 1.2 you'd have
crossed a chunk bound. During that particular sample of output you could lerp
between chunk 0 in the array and chunk 1 in the array. I don't think that such
levels of accuracy are really necessary to produce a nice DMG sound, but it's
something you should consider for yourself when you're making your own synth.

### Effect Timer

So let's make a struct for tracking an effect timer. This one is pretty simple.

* **Use Case:** We want to construct one of these by saying something like
  "overflow every 3/128ths of a second" or a similar value. We measure time in
  samples per second so we need to calibrate it with that too.

```rust
  let sweep_timer = EffectTimer::every(3, 128, samples_per_second);
```

* **Use Case:** Every time through a sample we'll tick all the appropriate
  effect timers and do the effects if they trigger, so the tick method will
  return a bool of if there was an overflow on that timer or not.

```rust
  if envelope_timer.tick() {
    // TODO: adjust the envelope settings here
  }
```

Seem easy to use? I think so, I hope you also think so.

So we have a struct like this:

```rust
pub struct EffectTimer {
  pub position: f32,
  pub cycle_per_tick: f32
}
```

And some methods that look like this:

```rust
impl EffectTimer {
  pub fn every(num: u32, denom: u32, samples_per_second: u32) -> Self {
    let time_per_effect = num as f32 / denom as f32;
    let samples_per_effect = time_per_effect * samples_per_second as f32;
    let position_per_tick = 1.0 / samples_per_effect;
    debug_assert!(position_per_tick < 1.0);
    Self {
      position: 0.0,
      position_per_tick,
    }
  }

  pub fn tick(&mut self) -> bool {
    self.position += self.position_per_tick;
    if self.position >= 1.0 {
      self.position -= 1.0;
      true
    } else {
      false
    }
  }
}
```

does it work? We can write a test to hopefully see.

```rust
#[test]
pub fn effect_timer_test() {
  const SAMPLES_PER_SECOND: u32 = 48_000;
  for &denom in &[64, 128, 256] {
    for num in 1..=7 {
      let mut timer = EffectTimer::every(num, denom, SAMPLES_PER_SECOND);
      let mut rollovers = 0;
      for _ in 0..(SAMPLES_PER_SECOND * num) {
        if timer.tick() {
          rollovers += 1;
        }
      }
      assert!(rollovers == denom || rollovers + 1 == denom);
    }
  }
}
```

Ah, and you might be wondering what's up with that assert. Yeah, because of
floating point precision issues we won't always have our triggers fire quite
when we want. They end up being a little bit slower than the correct frequency.
Even if we switch to using `f64` instead of `f32` we'll have to face this fact.

This basically happens because our output sample rate can't divide evenly into
the sample rate. The DMG didn't suffer from this because it had the sound chip
wired directly to the speakers. It didn't need to buffer samples, it just played
them immediately. That allowed it to have an effective samples per second rate
of **1,048,576**. That's _really_ high, but more importantly it divides evenly
by 64, 128, and 256, so you can track the effect periods over an even number of
samples.

* We could double our output frequency (to 96k) and then we'd also be at an even
  multiple of 256. Then we can use just use integers for everything and track
  time in terms of 1/256th of a second units (375 samples). Eg: a 7/128th of a
  second Sweep interval can be tracked with a 1/256th second timer and then you
  wait for it to roll over 14 times.
* We could try to set up some fixed point math with enough fractional precision
  so that we can always track small enough values to run whatever interval
  effect timer we need even with a sample rate of only 48k.
* We could just accept some slop and not worry about it.

For now we'll just accept the slop and not worry about it too much. As long as
we're aware of where we're getting it wrong it's not so bad, and we can always
choose to improve it later if we need to.

### Wave Form Timer

The wave form timer is the more complicated timer. We still tick forward by one
sample at a time, but instead of getting a `bool` back saying if there was an
overflow or not, we get back some information about which chunk values we should
use to determine our output sample.

What exactly does that information look like? That's a good question. I think
we'll want our output format to be the starting chunk as well as a count of how
many chunks after that we should also average in.

```rust
let (target_chunk, chunk_bounds_crossed) = wave_form_timer.tick();
let sample_value = if chunk_bounds_crossed > 1 {
  // average many chunks
} else {
  // check just one chunk, or possibly interpolate two if you really want
};
```

Does that look okay? I think it looks okay. How do we make one of these things?

```rust
let chunk_timer = ChunkTimer::for_wave(wave.len(), wave_hz, sample_rate);
```

Seems easy enough? Yeah probably. Okay let's do the code for that.

```rust
pub struct ChunkTimer {
  pub wave_position: f32,
  pub chunks_per_sample: f32,
  pub chunk_count: usize,
}
impl ChunkTimer {
  pub fn for_wave(chunk_count: usize, frequency: u32, sample_rate: u32) -> Self {
    let chunks_per_sample = chunk_count as f32 / (sample_rate as f32 / frequency as f32);
    Self {
      wave_position: 0.0,
      chunks_per_sample,
      chunk_count,
    }
  }

  pub fn tick(&mut self) -> (usize, usize) {
    let chunk_count_f = self.chunk_count as f32;
    let base_index = self.wave_position as usize;
    self.wave_position += self.chunks_per_sample;
    let final_index = self.wave_position as usize;
    if self.wave_position >= chunk_count_f {
      if self.wave_position >= chunk_count_f * 2.0 {
        self.wave_position %= chunk_count_f;
      } else {
        self.wave_position -= chunk_count_f;
      }
    }
    let chunk_bounds_crossed = final_index - base_index;
    (base_index, chunk_bounds_crossed)
  }
}
```

And a quick test as well:

```rust
#[test]
pub fn chunk_timer_test() {
  const SAMPLES_PER_SECOND: u32 = 48_000;
  for &frequency in &[32, 261, 6000, 50_001] {
    for &length in &[1, 8, 32, 33] {
      let mut chunk_timer = ChunkTimer::for_wave(length, frequency, SAMPLES_PER_SECOND);
      let (start_index, first_chunk_bounds_crossed) = chunk_timer.tick();
      assert_eq!(start_index, 0);
      for _ in 0..100 {
        let (base_index, new_chunk_bounds_crossed) = chunk_timer.tick();
        assert!(base_index < length);
        assert!(
          new_chunk_bounds_crossed == first_chunk_bounds_crossed
            || new_chunk_bounds_crossed == first_chunk_bounds_crossed + 1
        );
      }
    }
  }
}
```

Alright, so we can keep track of time as it passes. I guess we need to implement
at least one voice and see if these timers let us do that well.

Which voice is first? Well, _The Ultimate Game Boy Talk_ describes the Wave
voice first and calls it the simplest to understand. Having done this once
before, I agree with that assessment, and we'll go with that first.

## Wave Voice

TODO
