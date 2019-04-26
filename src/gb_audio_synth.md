
Recently a thing called [GB Studio](https://www.gbstudio.dev/) came out that
revived a bit of public interest in the Game Boy as a system. It's neat. Like
having RPG Maker for the Game Boy. You probably couldn't use it to make a super
high complexity Game Boy game, the Game Boy hardware is really just _so_ limited
you really need to be programming in ASM. Still, for beginners who just want to
try things out it's always better to have it than not have it.

Well, in that spirit, what if we wanted to reproduce some elements of the Game
Boy experience without living by the _actual_ Game Boy hardware limits?
Something like Shovel Knight, or Sonic Mania, where you start with the look and
sound of an old console game, and then [break past the real
limit](https://yachtclubgames.com/2014/07/breaking-the-nes/) of what the
hardware at the time could do.

# Game Boy Audio Synthesis

We're gonna be making some sounds that sound like the sounds the Game Boy made.

We'll be making a system that has an authentic sound output, but is easy to
control in the style of a modern library. It obviously will be brazenly
wasteful, using entire _bytes_ to store a number when just 2 bits could have
done the job. Yet, despite this, I think it will turn out great.

* **Just to be clear from the start:** this will _not_ end with a system ready
  made for use as part of of a Game Boy emulator. We won't bother to follow the
  exact memory layout that the Game Boy had, and we won't bother to be accurate
  down to the cycle with our sounds. If you'd like to actually make an emulator
  I think this will still be a helpful explanation to read, but you'll have to
  be stricter about how you implement things in your emulator's version.

I will be writing the actual code in Rust, but the theory is the same for any
programming language, and I will attempt to keep my use of any Rust magic to a
relative minimum so that programmers from other languages can also follow along.

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

Let's define just a few terms before we go on:

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
* **Samples Per Second / Sample Rate:** If a sample is like a pixel for sound,
  your samples _per second_ is like your pixels per inch in a picture. Just like
  how more pixels per inch gives a clearer image, more samples per second gives
  a clearer sound. There are [many common
  frequencies](https://en.wikipedia.org/wiki/Sampling_(signal_processing)#Sampling_rate).
  You don't want to go overboard if the situation doesn't call for it: the down
  side to having more samples per second is that it takes more time to compute
  the sound and it takes more memory to store. We'll use 48,000 Hz as our sample
  rate, which is the "basic" level of professional quality.
* **Voice:** In _The Ultimate Game Boy Talk_ the term
  "[voice](https://www.rolandcorp.com.au/blog/a-to-z-synthesizer#SGVoice)" is
  used to mean each of the four types of sounds that a Game Boy could make.
  That's a common enough term in synthesizer stuff, so we'll just stick with
  calling each type of sound a "voice": Pulse A, Pulse B, Wave, and Noise.

## Our Output Target

Obviously there's many sound APIs available, but they all fundamentally involve
filling a buffer that's calibrated to a given samples per second with sound
samples, then those sound sample go out to the sound card which plays them on
the speaker. Or maybe they're saved to disk for playback later.

Assuming that we're producing sound out to the sound card on the fly, we might
need to fill the sound buffer in a callback that the sound card executes when
the speakers are about to run out of samples, or we might need to enqueue
samples ourselves at some regular rate. It's actually fairly uninteresting which
style is used, so we will only concern ourselves here with the idea of filling a
sound buffer that we've been assigned to fill. When exactly that happens is for
somebody else to think about.

We'll assume that the caller is giving us two values:

* The sample rate of the buffer to fill
* The buffer to fill

Since we'll have to definitely keep our left channel and right channel clear,
let's make a struct for a single sample with `left` and `right` fields:

```rust
/// A single moment of stereo sound.
#[derive(Debug, Clone, Copy, Default)]
#[repr(C)]
pub struct StereoI16 {
  pub left: i16,
  pub right: i16,
}
```

The `repr(C)` means it's laid out in memory just as C would do it, in order of
the fields named. Also, I chose to list `left` before `right` because that's the
channel ordering that SDL2 uses. Obviously you should flip that around if your
program's native sound API uses an inverted channel ordering.

Okay, now let's look at what the top-level function signature for our sound
output might be:

```rust
/// Uses the Audio Processing Unit (APU) config to fill the sound buffer.
///
/// Naturally, this changes the state of the APU, as timers tick up, reset,
/// change various fields, etc.
pub fn make_the_sound(config: &mut APU, sample_rate: i32, buffer: &mut [StereoI16]) {
  unimplemented!("TODO: all of it")
}
```
