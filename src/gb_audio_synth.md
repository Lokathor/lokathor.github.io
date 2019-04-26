
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

* **Just to be clear from the start:** this will _not_ be a system ready made
  for use as part of of a Game Boy emulator. We won't bother to follow the exact
  memory layout that the Game Boy had, and we won't bother to be accurate down
  to the cycle with our sounds. If you'd like to make an emulator I think this
  will still be a helpful explanation to read, but you'll have to be stricter
  about how you implement things in your emulator's version.

We'll be making a system that has an authentic sound output, but is easy to
control in the style of a modern library. It obviously will be brazenly
wasteful, using entire _bytes_ to store a number when just 2 bits could have
done the job. Yet, despite this, I think it will turn out great.

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

I don't expect you to have read all those articles, but I do expect you to have
watched the sound portion of _The Ultimate Game Boy Talk_ at least once (it's
only about 2 minutes).

## Some Terms

Let's define just a few terms before we go on:

* **DMG:** The Game Boy's production identifier was DMG for "Dot-Matrix Game".
  I've been trying to not type "DMG" before this point in the document and it's
  already been a bit hard. I'll probably say DMG all over the place by the time
  we're done.
* **Channel:** I am personally using SDL2 for audio output in my "scratch
  project" where I've built this and other things. They use
  "[channel](https://wiki.libsdl.org/SDL_AudioSpec)" to mean the number of
  output locations (mono, stereo, quad, or 5.1) for the sound stream, so we'll
  also use "channel" to mean the same thing to keep confusion to a minimum. The
  Game Boy couldn't actually produce different sounds in the Left and Right
  channels, but you could at least adjust the volumes for each side separately,
  so we'll be outputting stereo sound.
* **Voice:** In _The Ultimate Game Boy Talk_ the term
  "[voice](https://www.rolandcorp.com.au/blog/a-to-z-synthesizer#SGVoice)" is
  used to mean each of the four types of sounds that a Game Boy could make.
  That's a common enough term in synthesizer stuff, so we'll just stick with
  calling each type of sound a "voice": Pulse A, Pulse B, Wave, and Noise.

